<template>
    <div v-if="slots.length > 0" class="block-container block-container--padded flavor-alternatives">
        <h3 class="block-container__title">Flavor matches on your shelf</h3>
        <p class="flavor-alternatives__intro">
            Bottles ranked by how well they fit each constrained slot.
            <em>Squarely in pattern</em> = penalty 0.
            <em>Slight stray</em> = penalty &lt; 2.
            <em>Off-pattern (hard limit)</em> = a hard constraint disqualified it.
        </p>
        <div v-for="slot in slots" :key="slot.sort" class="flavor-alternatives__slot">
            <div class="flavor-alternatives__slot-header">
                <strong>{{ slot.category }} slot</strong>
                <span class="flavor-alternatives__current">currently {{ slot.recipe_ingredient.name }}</span>
                <span v-if="slot.also_accept_categories.length > 0" class="flavor-alternatives__also-accept">
                    + accepts {{ slot.also_accept_categories.join(', ') }}
                </span>
            </div>
            <ol class="flavor-alternatives__list">
                <li v-for="(alt, idx) in slot.alternatives" :key="idx" class="flavor-alternatives__alt">
                    <span class="flavor-alternatives__rank">{{ idx + 1 }}.</span>
                    <span class="flavor-alternatives__name">{{ alt.bottle.name }}</span>
                    <span class="flavor-alternatives__penalty" :class="verdictClass(alt)">
                        {{ alt.verdict }} (penalty {{ alt.penalty }})
                    </span>
                    <span v-if="alt.bottle.confidence" class="flavor-alternatives__confidence">[{{ alt.bottle.confidence }}]</span>
                    <ul v-if="alt.flags.length > 0" class="flavor-alternatives__flags">
                        <li v-for="(flag, fi) in alt.flags" :key="fi">{{ flag }}</li>
                    </ul>
                </li>
            </ol>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'
import BarAssistantClient from '@/api/BarAssistantClient'

interface CocktailIngredient {
    sort: number
    ingredient: { id: number; name: string }
}

interface RecipeIngredient {
    id: number
    name: string
}

interface AlternativeRow {
    bottle: { id: number; name: string; category: string; confidence: string | null }
    penalty: number
    disqualified: boolean
    verdict: string
    flags: string[]
    cross_category: boolean
}

interface SlotResult {
    sort: number
    ingredient_name: string
    recipe_ingredient: RecipeIngredient
    category: string
    also_accept_categories: string[]
    alternatives: AlternativeRow[]
}

const props = defineProps<{
    cocktailId: number
    ingredients: CocktailIngredient[]
}>()

const slots = ref<SlotResult[]>([])

function verdictClass(alt: AlternativeRow): string {
    if (alt.disqualified) return 'flavor-alternatives__penalty--off'
    if (alt.penalty === 0) return 'flavor-alternatives__penalty--in'
    if (alt.penalty <= 2) return 'flavor-alternatives__penalty--slight'
    return 'flavor-alternatives__penalty--notable'
}

async function load() {
    if (!props.cocktailId || props.ingredients.length === 0) {
        slots.value = []
        return
    }
    const results: SlotResult[] = []
    // Probe each ingredient's slot in parallel.
    await Promise.all(
        props.ingredients.map(async (ing) => {
            try {
                const resp = (await BarAssistantClient.getSlotAlternatives(
                    props.cocktailId,
                    ing.sort,
                    { on_shelf_only: true, include_strays: true, top_n: 5 },
                )) as { data?: SlotResult }
                // Filter out the recipe's own ingredient — "alternatives" means "instead of."
                const filtered = (resp?.data?.alternatives ?? []).filter(
                    (a) => a.bottle.id !== ing.ingredient.id,
                )
                if (filtered.length) {
                    results.push({
                        sort: ing.sort,
                        ingredient_name: ing.ingredient.name,
                        recipe_ingredient: { id: ing.ingredient.id, name: ing.ingredient.name },
                        category: (resp.data as any).category,
                        also_accept_categories: (resp.data as any).also_accept_categories ?? [],
                        alternatives: filtered,
                    })
                }
            } catch (e: unknown) {
                // 404 = no slot meta declared for this slot — skip silently.
                const err = e as { response?: { status?: number } }
                if (err?.response?.status !== 404) {
                    console.warn('FlavorAlternatives: slot failed', ing.sort, e)
                }
            }
        }),
    )
    results.sort((a, b) => a.sort - b.sort)
    slots.value = results
}

onMounted(load)
watch(() => [props.cocktailId, props.ingredients.map(i => i.sort).join(',')], load)
</script>

<style scoped>
.flavor-alternatives {
    margin-top: 1rem;
}
.flavor-alternatives__intro {
    font-size: 0.85rem;
    color: var(--clr-text-secondary, #888);
    margin: 0.25rem 0 0.75rem;
}
.flavor-alternatives__slot {
    margin: 1rem 0;
}
.flavor-alternatives__slot-header {
    margin-bottom: 0.5rem;
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
    flex-wrap: wrap;
}
.flavor-alternatives__slot-header strong {
    text-transform: capitalize;
}
.flavor-alternatives__current {
    font-size: 0.85rem;
    color: var(--clr-text-secondary, #888);
}
.flavor-alternatives__also-accept {
    font-size: 0.8rem;
    color: var(--clr-text-secondary, #888);
}
.flavor-alternatives__list {
    list-style: none;
    padding: 0;
    margin: 0;
}
.flavor-alternatives__alt {
    padding: 0.4rem 0;
    border-bottom: 1px solid var(--clr-border, #eee);
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 0.5rem;
}
.flavor-alternatives__alt:last-child { border-bottom: none; }
.flavor-alternatives__rank {
    width: 1.5rem;
    color: var(--clr-text-secondary, #888);
}
.flavor-alternatives__name {
    flex: 1 1 auto;
    min-width: 12rem;
    font-weight: 500;
}
.flavor-alternatives__penalty {
    font-size: 0.85rem;
    padding: 0.1rem 0.4rem;
    border-radius: 3px;
}
.flavor-alternatives__penalty--in {
    background: rgba(0, 200, 100, 0.15);
    color: #198754;
}
.flavor-alternatives__penalty--slight {
    background: rgba(255, 193, 7, 0.15);
    color: #8a6d3b;
}
.flavor-alternatives__penalty--notable {
    background: rgba(255, 150, 50, 0.18);
    color: #b8651b;
}
.flavor-alternatives__penalty--off {
    background: rgba(220, 53, 69, 0.15);
    color: #c44a4a;
}
.flavor-alternatives__confidence {
    font-size: 0.8rem;
    color: var(--clr-text-secondary, #888);
}
.flavor-alternatives__flags {
    list-style: none;
    padding: 0;
    margin: 0.25rem 0 0 2rem;
    font-size: 0.8rem;
    color: var(--clr-text-secondary, #888);
    flex-basis: 100%;
}
</style>
