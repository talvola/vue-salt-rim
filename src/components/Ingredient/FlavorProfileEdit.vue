<template>
    <SaltRimDialog v-model="open">
        <template #trigger="{ toggleDialog }">
            <button type="button" class="button button--outline flavor-edit__trigger" @click.prevent="toggleDialog()">
                {{ existingProfile ? 'Edit flavor profile' : 'Add flavor profile' }}
            </button>
        </template>
        <template #dialog="{ toggleDialog }">
            <h3>{{ existingProfile ? 'Edit' : 'Add' }} flavor profile</h3>
            <form @submit.prevent="save(toggleDialog)" class="flavor-edit__form">
                <div class="form-group">
                    <label for="flavor-cat">Category</label>
                    <select id="flavor-cat" v-model="category" :disabled="categories.length === 0" @change="onCategoryChange" required>
                        <option value="" disabled>Select a category</option>
                        <option v-for="c in categories" :key="c.category" :value="c.category">{{ c.category }}</option>
                    </select>
                </div>

                <div v-for="axis in axes" :key="axis" class="form-group flavor-edit__axis">
                    <label :for="`axis-${axis}`">{{ axis }}</label>
                    <input
                        :id="`axis-${axis}`"
                        type="range"
                        min="0" max="3" step="1"
                        v-model.number="profile[axis]"
                        class="flavor-edit__slider"
                    />
                    <span class="flavor-edit__value">{{ profile[axis] ?? 0 }}</span>
                </div>

                <div class="form-group">
                    <label for="flavor-source">Source</label>
                    <select id="flavor-source" v-model="source">
                        <option value="">—</option>
                        <option value="tgii">tgii</option>
                        <option value="llm_from_description">llm_from_description</option>
                        <option value="manual">manual</option>
                        <option value="manual_override">manual_override</option>
                    </select>
                </div>

                <div class="form-group">
                    <label for="flavor-conf">Confidence</label>
                    <select id="flavor-conf" v-model="confidence">
                        <option value="">—</option>
                        <option value="high">high</option>
                        <option value="medium">medium</option>
                        <option value="low">low</option>
                    </select>
                </div>

                <div class="form-group">
                    <label for="flavor-notes">Notes</label>
                    <textarea id="flavor-notes" v-model="notes" rows="3"></textarea>
                </div>

                <div class="form-group">
                    <label class="flavor-edit__checkbox">
                        <input type="checkbox" v-model="suggestableForClassics" />
                        Suggestable for classic cocktails
                    </label>
                    <small class="flavor-edit__hint">
                        Untick for novelty/joke bottles (e.g. Crab Trapper) so the matcher never surfaces them
                        even if their profile coincidentally fits.
                    </small>
                </div>

                <div v-if="error" class="flavor-edit__error">{{ error }}</div>

                <div class="dialog-buttons">
                    <button type="button" class="button button--outline" @click="toggleDialog()">Cancel</button>
                    <button type="submit" class="button button--primary" :disabled="saving || !category">
                        {{ saving ? 'Saving…' : 'Save' }}
                    </button>
                </div>
            </form>
        </template>
    </SaltRimDialog>
</template>

<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'
import BarAssistantClient from '@/api/BarAssistantClient'
import SaltRimDialog from '@/components/Dialog/SaltRimDialog.vue'

interface ExistingProfile {
    ingredient_id: number
    category: string
    profile: Record<string, number>
    source: string | null
    confidence: string | null
    notes: string | null
    suggestable_for_classics: boolean
}

interface Category {
    category: string
    axes: string[]
}

const props = defineProps<{
    ingredientId: number
    existingProfile: ExistingProfile | null
}>()

const emit = defineEmits<{ (e: 'saved'): void }>()

const open = ref(false)
const categories = ref<Category[]>([])
const category = ref('')
const axes = ref<string[]>([])
const profile = ref<Record<string, number>>({})
const source = ref('')
const confidence = ref('')
const notes = ref('')
const suggestableForClassics = ref(true)
const saving = ref(false)
const error = ref('')

async function loadCategories() {
    try {
        const resp = await BarAssistantClient.getFlavorCategories()
        categories.value = resp?.data ?? []
    } catch (e) {
        console.warn('FlavorProfileEdit: failed to load categories', e)
    }
}

function onCategoryChange() {
    const cat = categories.value.find(c => c.category === category.value)
    axes.value = cat?.axes ?? []
    // Seed missing axes with 0 if the user just switched category
    for (const axis of axes.value) {
        if (!(axis in profile.value)) {
            profile.value[axis] = 0
        }
    }
    // Drop axes from prior category that aren't in the new one
    for (const axis of Object.keys(profile.value)) {
        if (!axes.value.includes(axis)) {
            delete profile.value[axis]
        }
    }
}

function hydrateFromExisting() {
    if (props.existingProfile) {
        category.value = props.existingProfile.category
        profile.value = { ...props.existingProfile.profile }
        source.value = props.existingProfile.source ?? ''
        confidence.value = props.existingProfile.confidence ?? ''
        notes.value = props.existingProfile.notes ?? ''
        suggestableForClassics.value = props.existingProfile.suggestable_for_classics
        const cat = categories.value.find(c => c.category === category.value)
        axes.value = cat?.axes ?? Object.keys(props.existingProfile.profile)
    } else {
        category.value = ''
        axes.value = []
        profile.value = {}
        source.value = 'manual'
        confidence.value = ''
        notes.value = ''
        suggestableForClassics.value = true
    }
}

async function save(toggleDialog: () => void) {
    saving.value = true
    error.value = ''
    try {
        const body: any = {
            category: category.value,
            profile: { ...profile.value },
            suggestable_for_classics: suggestableForClassics.value,
        }
        if (source.value) body.source = source.value
        if (confidence.value) body.confidence = confidence.value
        if (notes.value) body.notes = notes.value
        await BarAssistantClient.putIngredientFlavorProfile(props.ingredientId, body)
        emit('saved')
        toggleDialog()
    } catch (e: any) {
        error.value = e?.body?.message ?? e?.message ?? 'Save failed'
    } finally {
        saving.value = false
    }
}

watch(open, (v) => {
    if (v) hydrateFromExisting()
})

onMounted(async () => {
    await loadCategories()
    hydrateFromExisting()
})
</script>

<style scoped>
.flavor-edit__trigger {
    margin-top: 0.5rem;
}
.flavor-edit__form {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
}
.flavor-edit__axis {
    display: grid;
    grid-template-columns: 6rem 1fr 2rem;
    align-items: center;
    gap: 0.5rem;
}
.flavor-edit__axis label {
    text-transform: capitalize;
    margin: 0;
}
.flavor-edit__slider {
    width: 100%;
}
.flavor-edit__value {
    font-weight: bold;
    text-align: right;
}
.flavor-edit__checkbox {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    font-weight: normal;
    margin: 0;
}
.flavor-edit__hint {
    display: block;
    margin-top: 0.25rem;
    color: var(--clr-text-secondary, #888);
}
.flavor-edit__error {
    color: #c44a4a;
    background: rgba(196, 74, 74, 0.1);
    padding: 0.5rem;
    border-radius: 4px;
}
.dialog-buttons {
    display: flex;
    gap: 0.5rem;
    justify-content: flex-end;
    margin-top: 1rem;
}
</style>
