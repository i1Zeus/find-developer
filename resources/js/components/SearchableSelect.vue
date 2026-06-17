<script setup lang="ts">
import { useDebounceFn } from '@vueuse/core';
import { Check, ChevronsUpDown, X } from 'lucide-vue-next';
import { computed, onMounted, ref, useId, watch } from 'vue';
import { Badge } from '@/components/ui/badge';
import {
    Combobox,
    ComboboxAnchor,
    ComboboxCancel,
    ComboboxEmpty,
    ComboboxGroup,
    ComboboxInput,
    ComboboxItem,
    ComboboxItemIndicator,
    ComboboxList,
    ComboboxTrigger,
    ComboboxViewport,
} from '@/components/ui/combobox';
import { cn } from '@/lib/utils';

const instanceId = useId();

const props = withDefaults(
    defineProps<{
        modelValue?: string | string[] | null;
        multiple?: boolean;
        open?: boolean;
        options?: { value: string; label: string }[];
        optionsUrl?: string;
        placeholder?: string;
        id?: string;
        class?: string;
        maxOptions?: number;
        allowClear?: boolean;
    }>(),
    {
        modelValue: null,
        multiple: false,
        options: () => [],
        placeholder: 'Search...',
        maxOptions: 50,
        allowClear: true,
    },
);

const emit = defineEmits<{
    (e: 'update:modelValue', value: string | string[] | null): void;
    (e: 'update:open', value: boolean): void;
}>();

const searchTerm = ref('');
const fetchedOptions = ref<{ value: string; label: string }[]>([]);
const optionsLoading = ref(false);
const internalOpen = ref(false);

const effectiveOpen = computed(() =>
    props.open !== undefined ? props.open : internalOpen.value,
);

function handleOpenChange(open: boolean): void {
    if (props.open === undefined) {
        internalOpen.value = open;
    }
    if (open && props.optionsUrl) {
        fetchOptions();
    }
    emit('update:open', open);
}

const selectedValues = computed(() => {
    const val = props.modelValue;
    if (val == null) return [];
    return Array.isArray(val) ? val : [val];
});

// Cache for storing selected options' labels so they are not lost when searching or filtering
const selectedOptionsCache = ref<Record<string, string>>({});

watch(
    [selectedValues, () => props.options, fetchedOptions],
    () => {
        selectedValues.value.forEach((v) => {
            const strVal = String(v);

            // Try to find the label in props.options first
            const opt = props.options?.find((o) => String(o.value) === strVal);
            if (opt) {
                selectedOptionsCache.value[strVal] = opt.label;
                return;
            }

            // Otherwise try to find in fetchedOptions
            const fetched = fetchedOptions.value.find(
                (o) => String(o.value) === strVal || String(o.label) === strVal,
            );
            if (fetched) {
                selectedOptionsCache.value[strVal] = fetched.label;
            }
        });
    },
    { immediate: true },
);

const selectedOptions = computed(() =>
    selectedValues.value.map((v) => {
        const strVal = String(v);
        return {
            value: v,
            label: selectedOptionsCache.value[strVal] ?? strVal,
        };
    }),
);

const displayValue = computed(() => {
    if (props.multiple && selectedOptions.value.length > 0) {
        return selectedOptions.value.map((o) => o.label).join(', ');
    }
    return selectedOptions.value[0]?.label ?? '';
});

const displayOptions = computed(() => {
    if (props.optionsUrl) {
        if (props.multiple) {
            // If multiple, exclude checked/selected items from dropdown completely
            return fetchedOptions.value
                .filter(
                    (o) =>
                        !selectedValues.value.some(
                            (v) => String(v) === String(o.value),
                        ),
                )
                .slice(0, props.maxOptions);
        } else {
            // For single select:
            // If searching, do NOT show the checked ones unless they match the search query (i.e. are in fetchedOptions)
            if (searchTerm.value) {
                return fetchedOptions.value.slice(0, props.maxOptions);
            } else {
                // Prepend selected option when not searching so user can see it in list
                const selected = selectedValues.value
                    .map((v) => {
                        const strVal = String(v);
                        const opt = fetchedOptions.value.find(
                            (o) =>
                                String(o.value) === strVal ||
                                String(o.label) === strVal,
                        );
                        return (
                            opt ?? {
                                value: v,
                                label:
                                    selectedOptionsCache.value[strVal] ??
                                    strVal,
                            }
                        );
                    })
                    .filter((o) => o.value);
                const fromFetched = fetchedOptions.value.filter(
                    (o) =>
                        !selected.some(
                            (s) => String(s.value) === String(o.value),
                        ),
                );
                return [...selected, ...fromFetched].slice(0, props.maxOptions);
            }
        }
    }

    if (props.multiple) {
        // Static options (multiple): filter out checked items
        return (props.options ?? [])
            .filter(
                (o) =>
                    !selectedValues.value.some(
                        (v) => String(v) === String(o.value),
                    ),
            )
            .slice(0, props.maxOptions);
    }

    return (props.options ?? []).slice(0, props.maxOptions);
});

async function fetchOptions(): Promise<void> {
    if (!props.optionsUrl) return;
    optionsLoading.value = true;
    try {
        const url = new URL(props.optionsUrl, window.location.origin);
        if (searchTerm.value) {
            url.searchParams.set('search', searchTerm.value);
            url.searchParams.set('filter[search]', searchTerm.value);
        }
        const res = await fetch(url.toString());
        if (!res.ok) throw new Error('Failed to fetch options');
        const json = await res.json();
        const items = Array.isArray(json.data) ? json.data : [];
        fetchedOptions.value = items.map((item: any) => {
            if (item && typeof item === 'object') {
                if ('value' in item && 'label' in item) {
                    return {
                        value: String((item as { value: unknown }).value),
                        label: String((item as { label: unknown }).label),
                    };
                }

                if ('id' in item && 'name' in item) {
                    return {
                        value: String((item as { id: unknown }).id),
                        label: String((item as { name: unknown }).name),
                    };
                }
            }

            const anyItem = item as {
                id?: unknown;
                name?: unknown;
                value?: unknown;
                label?: unknown;
            };
            const fallbackValue =
                anyItem.id ??
                anyItem.value ??
                anyItem.name ??
                anyItem.label ??
                '';
            const fallbackLabel =
                anyItem.name ??
                anyItem.label ??
                anyItem.value ??
                anyItem.id ??
                '';

            return {
                value: String(fallbackValue),
                label: String(fallbackLabel),
            };
        });
    } catch {
        fetchedOptions.value = [];
    } finally {
        optionsLoading.value = false;
    }
}

const debouncedFetch = useDebounceFn(() => {
    if (props.optionsUrl) fetchOptions();
}, 300);

watch(
    () => searchTerm.value,
    () => {
        if (props.optionsUrl) debouncedFetch();
    },
);

onMounted(() => {
    if (props.optionsUrl) fetchOptions();
});

function removeValue(value: string, event: Event): void {
    event.stopPropagation();
    const next = selectedValues.value.filter((v) => v !== value);
    emit('update:modelValue', next.length > 0 ? next : []);
}
</script>

<template>
    <Combobox :name="id ?? instanceId" :model-value="multiple ? selectedValues : (selectedValues[0] ?? null)"
        :multiple="multiple" :open="effectiveOpen" :open-on-click="true" :ignore-filter="!!optionsUrl"
        :reset-search-term-on-select="false" @update:model-value="
            emit(
                'update:modelValue',
                multiple
                    ? Array.isArray($event)
                        ? $event
                        : []
                    : Array.isArray($event)
                        ? ($event[0] ?? null)
                        : ($event ?? null),
            )
            " @update:open="handleOpenChange">
        <ComboboxAnchor :class="cn(
            'flex h-auto min-h-9 w-full items-center justify-between gap-2 rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm transition-colors',
            'focus-within:ring-1 focus-within:ring-ring focus-within:outline-none',
            props.class,
        )
            ">
            <ComboboxTrigger :id="id" as-child
                class="flex min-w-0 flex-1 cursor-pointer items-center gap-2 outline-none">
                <button type="button"
                    class="flex min-w-0 flex-1 cursor-pointer items-center gap-2 bg-transparent text-left outline-none">
                    <span v-if="multiple && selectedOptions.length > 0"
                        class="flex flex-1 flex-wrap items-center gap-1">
                        <Badge v-for="(opt, i) in selectedOptions" :key="`${instanceId}-badge-${opt.value}-${i}`"
                            variant="secondary" class="gap-1 pr-1 font-normal">
                            {{ opt.label }}
                            <button v-if="allowClear" type="button"
                                class="rounded-full ring-offset-background outline-none hover:bg-secondary focus:ring-2 focus:ring-ring focus:ring-offset-2"
                                aria-label="Remove" @click.stop="removeValue(opt.value, $event)">
                                <X class="size-3.5" />
                            </button>
                        </Badge>
                    </span>
                    <span v-else :class="cn(
                        'flex-1 truncate text-left',
                        !displayValue && 'text-muted-foreground',
                    )
                        ">
                        {{ displayValue || placeholder }}
                    </span>
                    <ChevronsUpDown class="size-4 shrink-0 opacity-50" />
                </button>
            </ComboboxTrigger>
            <ComboboxCancel class="sr-only" />
        </ComboboxAnchor>

        <ComboboxList class="max-h-[300px] w-(--reka-combobox-trigger-width) p-0" align="start">
            <div class="relative">
                <ComboboxInput v-model="searchTerm" :placeholder="placeholder"
                    class="flex-1 rounded-none border-0 pr-10" />
            </div>
            <ComboboxViewport>
                <ComboboxEmpty>
                    {{ optionsLoading ? 'Searching...' : 'No results found.' }}
                </ComboboxEmpty>
                <ComboboxGroup>
                    <ComboboxItem v-for="(opt, i) in displayOptions" :key="`${instanceId}-item-${opt.value}-${i}`"
                        :value="opt.value" :text-value="opt.label">
                        <Check v-if="multiple" :class="cn(
                            'mr-2 size-4 shrink-0',
                            selectedValues.includes(opt.value)
                                ? 'opacity-100'
                                : 'opacity-0',
                        )
                            " />
                        <ComboboxItemIndicator v-else class="mr-2">
                            <Check class="size-4" />
                        </ComboboxItemIndicator>
                        {{ opt.label }}
                    </ComboboxItem>
                </ComboboxGroup>
            </ComboboxViewport>
        </ComboboxList>
    </Combobox>
</template>
