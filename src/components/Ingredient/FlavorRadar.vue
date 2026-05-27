<template>
    <div v-if="profile || loaded" class="block-container block-container--padded flavor-radar">
        <div class="flavor-radar__header">
            <h2 class="block-container__title">Flavor profile</h2>
            <div class="flavor-radar__meta">
                <span v-if="profile" class="flavor-radar__category">{{ profile.category }}</span>
                <span v-if="profile?.confidence" class="flavor-radar__confidence">
                    [{{ profile.confidence }}]
                </span>
                <FlavorProfileEdit :ingredient-id="ingredientId" :existing-profile="profile" @saved="load" />
            </div>
        </div>
        <p v-if="!profile" class="flavor-radar__empty">No flavor profile yet. Click "Add flavor profile" to score this bottle.</p>
        <svg v-if="profile" :viewBox="`0 0 ${size} ${size}`" class="flavor-radar__svg" :style="{ maxWidth: size + 'px' }">
            <!-- Concentric rings at value=1, 2, 3 -->
            <polygon
                v-for="ring in 3"
                :key="`ring-${ring}`"
                :points="ringPoints(ring)"
                class="flavor-radar__ring"
            />
            <!-- Axis spokes -->
            <line
                v-for="(axis, idx) in axes"
                :key="`spoke-${idx}`"
                :x1="cx"
                :y1="cy"
                :x2="axisVertex(idx, maxValue).x"
                :y2="axisVertex(idx, maxValue).y"
                class="flavor-radar__spoke"
            />
            <!-- Data polygon -->
            <polygon :points="dataPoints" class="flavor-radar__data" />
            <!-- Vertices -->
            <circle
                v-for="(axis, idx) in axes"
                :key="`vertex-${idx}`"
                :cx="axisVertex(idx, profile.profile[axis] ?? 0).x"
                :cy="axisVertex(idx, profile.profile[axis] ?? 0).y"
                r="3"
                class="flavor-radar__vertex"
            />
            <!-- Axis labels -->
            <text
                v-for="(axis, idx) in axes"
                :key="`label-${idx}`"
                :x="labelPos(idx).x"
                :y="labelPos(idx).y"
                text-anchor="middle"
                dominant-baseline="middle"
                class="flavor-radar__label"
            >
                {{ axis }} <tspan class="flavor-radar__value">{{ profile.profile[axis] ?? 0 }}</tspan>
            </text>
        </svg>
        <p v-if="profile.notes" class="flavor-radar__notes">{{ profile.notes }}</p>
    </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import BarAssistantClient from '@/api/BarAssistantClient'
import FlavorProfileEdit from '@/components/Ingredient/FlavorProfileEdit.vue'

interface FlavorProfile {
    ingredient_id: number
    category: string
    profile: Record<string, number>
    source: string | null
    confidence: string | null
    notes: string | null
    scored_at: string | null
    suggestable_for_classics: boolean
}

interface Category {
    category: string
    axes: string[]
}

const props = defineProps<{ ingredientId: number }>()

const size = 280
const cx = size / 2
const cy = size / 2
const radius = (size / 2) - 40        // padding for labels
const maxValue = 3                     // axes are 0-3

const profile = ref<FlavorProfile | null>(null)
const categories = ref<Category[]>([])
const loaded = ref(false)

const axes = computed<string[]>(() => {
    if (!profile.value) return []
    const cat = categories.value.find(c => c.category === profile.value!.category)
    return cat?.axes ?? Object.keys(profile.value.profile)
})

function axisAngle(idx: number): number {
    // Start at top (12 o'clock) and go clockwise. Subtract π/2 to rotate.
    return (2 * Math.PI * idx / axes.value.length) - Math.PI / 2
}

function axisVertex(idx: number, value: number) {
    const r = (value / maxValue) * radius
    const a = axisAngle(idx)
    return { x: cx + r * Math.cos(a), y: cy + r * Math.sin(a) }
}

function ringPoints(ring: number): string {
    return axes.value
        .map((_, idx) => {
            const v = axisVertex(idx, ring)
            return `${v.x.toFixed(2)},${v.y.toFixed(2)}`
        })
        .join(' ')
}

const dataPoints = computed(() =>
    axes.value
        .map((axis, idx) => {
            const value = profile.value?.profile[axis] ?? 0
            const v = axisVertex(idx, value)
            return `${v.x.toFixed(2)},${v.y.toFixed(2)}`
        })
        .join(' '),
)

function labelPos(idx: number) {
    // Place label just outside the outer ring.
    const r = radius + 18
    const a = axisAngle(idx)
    return { x: cx + r * Math.cos(a), y: cy + r * Math.sin(a) }
}

async function load() {
    try {
        const [profRes, catsRes] = await Promise.all([
            BarAssistantClient.getIngredientFlavorProfile(props.ingredientId),
            BarAssistantClient.getFlavorCategories(),
        ])
        profile.value = profRes?.data ?? null
        categories.value = catsRes?.data ?? []
    } catch (e) {
        console.warn('FlavorRadar: failed to load', e)
        profile.value = null
    } finally {
        loaded.value = true
    }
}

onMounted(load)
watch(() => props.ingredientId, load)
</script>

<style scoped>
.flavor-radar {
    margin-top: 1rem;
}
.flavor-radar__header {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 0.5rem;
}
.flavor-radar__meta {
    font-size: 0.9rem;
    color: var(--clr-text-secondary, #888);
}
.flavor-radar__category {
    text-transform: capitalize;
    margin-right: 0.25rem;
}
.flavor-radar__svg {
    display: block;
    margin: 0 auto;
}
.flavor-radar__ring {
    fill: none;
    stroke: var(--clr-border, #ddd);
    stroke-width: 1;
}
.flavor-radar__spoke {
    stroke: var(--clr-border, #ddd);
    stroke-width: 1;
}
.flavor-radar__data {
    fill: var(--clr-accent, #c44a4a);
    fill-opacity: 0.25;
    stroke: var(--clr-accent, #c44a4a);
    stroke-width: 2;
}
.flavor-radar__vertex {
    fill: var(--clr-accent, #c44a4a);
}
.flavor-radar__label {
    font-size: 0.8rem;
    fill: var(--clr-text, #333);
    text-transform: capitalize;
}
.flavor-radar__value {
    font-weight: bold;
    margin-left: 0.2em;
}
.flavor-radar__notes {
    font-size: 0.85rem;
    color: var(--clr-text-secondary, #888);
    margin-top: 0.5rem;
    font-style: italic;
}
</style>
