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
            <!-- Axis labels — group adds a hover tooltip via <title> -->
            <g
                v-for="(axis, idx) in axes"
                :key="`label-${idx}`"
                class="flavor-radar__label-group"
            >
                <title>{{ axisDefinition(axis) }}</title>
                <text
                    :x="labelPos(idx).x"
                    :y="labelPos(idx).y"
                    text-anchor="middle"
                    dominant-baseline="middle"
                    class="flavor-radar__label"
                >
                    {{ axis }} <tspan class="flavor-radar__value">{{ profile.profile[axis] ?? 0 }}</tspan>
                </text>
            </g>
        </svg>
        <p v-if="profile && profile.notes" class="flavor-radar__notes">{{ profile.notes }}</p>
        <details v-if="profile" class="flavor-radar__legend">
            <summary>What do these axes mean?</summary>
            <dl class="flavor-radar__defs">
                <template v-for="axis in axes" :key="`def-${axis}`">
                    <dt>{{ axis }}</dt>
                    <dd>{{ axisDefinition(axis) }}</dd>
                </template>
            </dl>
        </details>
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

// Definitions for each axis name. Where an axis takes on a different meaning
// per category (e.g., `spice` means rye-pepper in bourbon vs warm baking spice
// in herbal liqueurs), the definition covers both registers in one sentence.
const AXIS_DEFS: Record<string, string> = {
    juniper: 'Juniper berry / pine-resin character — the defining note of London-Dry-style gin.',
    citrus: 'Bright citrus peel and zest — orange, lemon, lime, chinotto. Distinguishes Aperol from Averna, Lillet Blanc from Carpano.',
    floral: 'Flower aromatics — violet, rose, lavender, hibiscus, elderflower. High floral in gin can fight a Negroni; low floral keeps a Martini classic.',
    heat: 'Perceived burn / alcoholic intensity. Navy-strength gins push this high.',
    spice: 'Spice register. In gin: pepper / coriander warmth. In bourbon/rye: rye-grain pepper and baking spice. In herbal liqueurs: warm cinnamon-clove-allspice. In fruit liqueurs: spice-driven cordials (BroVo Boomerang) vs single-fruit (Maraschino).',
    herbal: 'Aromatic herbs — sage, thyme, alpine herbs, eucalyptus. Defining for Chartreuse, Strega, mountain amari.',
    fruited: 'Wine-cask / dried-fruit character. High fruited gins (Renais) or vermouths (Carpano) signal grape/berry weight on the palate.',
    fruit: 'Fruit esters and dried-fruit character — applies to whiskey (long-aging develops this) and some rums.',
    bitter: 'Bitterness intensity — gentian, cinchona, wormwood, hops. Aperol is low-bitter; Fernet-Branca and Suze hit the ceiling. Punt e Mes pushes vermouth into half-amaro territory.',
    sweet: 'Perceived sweetness from sugar / added sweetener. Wheated bourbons run sweet; dry vermouth and absinthe run low.',
    dark: 'Dark register — coffee, cola, chocolate, molasses, smoked-rhubarb. Averna and Ramazzotti are dark-forward; Aperol is not.',
    mint: 'Menthol / peppermint cooling. The Fernet family signature. A Negroni hard-caps this at zero.',
    root: 'Earthy roots and rhizomes — gentian (Suze), rhubarb (Sfumato, Zucca), walnut (Nocino-style), artichoke (Cynar).',
    oak: 'Oak-derived tannin and wood character from cask aging. El Dorado 15 and Mountain Summit run high.',
    vanilla: 'Sweet wood notes — vanilla, caramel, butterscotch. American new-oak aging is the source.',
    body: 'Mouthfeel weight — proof and oily-vs-light texture. Cask-strength whiskeys score high.',
    smoke: 'Peat / smoke. Defining for scotch (Ardbeg, Lagavulin = max) and American single malt (BBQ-wood smoked like Andalusia Stryker).',
    funk: 'Jamaican-style ester intensity — pineapple, banana, hogo. Smith & Cross and Worthy Park are the benchmark; Bacardi is zero.',
    molasses: 'Heavy molasses / black-strap weight. Demerara rums and Cruzan Black Strap score high; agricole is zero.',
    grassy: 'Cane-juice / fresh-grass character. The defining axis for rhum agricole and cachaça (Clement, Kuleana, Capi).',
    orchard: 'Tree-fruit family — cherry, apricot, peach, apple, pear, sloe. Maraschino is pure cherry; apricot liqueur is pure apricot.',
    berry: 'Berry family — blackcurrant (cassis), strawberry, raspberry, hibiscus.',
    tropical: 'Tropical fruit — passion fruit, melon, pineapple, mango.',
    anise: 'Anise / licorice character — Strega, Galliano, pastis, absinthe.',
    honey: 'Honeyed sweetness — Yellow Chartreuse, Benedictine, and Drambuie all share this register.',
    cooling: 'Cool / menthol / alpine / pine — Green Chartreuse, Agwa, Quaglia Pino Mugo all share this fresh-cool finish.',
}

function axisDefinition(axis: string): string {
    return AXIS_DEFS[axis] ?? '(definition pending)'
}

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
.flavor-radar__label-group {
    cursor: help;
}
.flavor-radar__legend {
    margin-top: 1rem;
    font-size: 0.9rem;
}
.flavor-radar__legend summary {
    cursor: pointer;
    color: var(--clr-text-secondary, #888);
    user-select: none;
}
.flavor-radar__defs {
    margin: 0.5rem 0 0;
    display: grid;
    grid-template-columns: 6rem 1fr;
    gap: 0.25rem 0.75rem;
}
.flavor-radar__defs dt {
    text-transform: capitalize;
    font-weight: 600;
    color: var(--clr-text, #333);
}
.flavor-radar__defs dd {
    margin: 0;
    color: var(--clr-text-secondary, #555);
}
</style>
