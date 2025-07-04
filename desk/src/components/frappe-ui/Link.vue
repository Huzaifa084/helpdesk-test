<template>
  <div class="space-y-1.5">
    <label class="block" :class="labelClasses" v-if="attrs.label">
      {{ attrs.label }}
    </label>
    <Autocomplete
      ref="autocomplete"
      :options="options.data"
      v-model="value"
      :size="attrs.size || 'sm'"
      :variant="attrs.variant"
      :placeholder="attrs.placeholder"
      :filterable="false"
    >
      <template #target="{ open, togglePopover }">
        <slot name="target" v-bind="{ open, togglePopover }" />
      </template>

      <template #prefix>
        <slot name="prefix" />
      </template>

      <template #item-prefix="{ active, selected, option }">
        <slot name="item-prefix" v-bind="{ active, selected, option }" />
      </template>

      <template #item-label="{ active, selected, option }">
        <slot name="item-label" v-bind="{ active, selected, option }">
          <div
            v-if="option.description && showDescription"
            class="flex flex-col gap-1"
          >
            <div class="flex-1 font-semibold truncate text-ink-gray-7">
              {{ option.label }}
            </div>
            <div class="flex-1 text-sm truncate text-ink-gray-5">
              {{ option.description }}
            </div>
          </div>
          <div v-else class="flex-1 truncate text-ink-gray-7">
            {{ option.label }}
          </div>
        </slot>
      </template>

      <template #footer="{ value, close }" v-if="!hideClearButton">
        <div v-if="attrs.onCreate">
          <Button
            variant="ghost"
            theme="blue"
            class="w-full !justify-start"
            :label="'Create New'"
            @click="attrs.onCreate(value, close)"
          >
            <template #prefix>
              <FeatherIcon name="plus" class="h-4" />
            </template>
          </Button>
        </div>
        <div>
          <Button
            variant="ghost"
            theme="blue"
            class="w-full !justify-start"
            :label="'Clear'"
            @click="() => clearValue(close)"
          >
            <template #prefix>
              <FeatherIcon name="x" class="h-4" />
            </template>
          </Button>
        </div>
      </template>
    </Autocomplete>
  </div>
</template>

<script setup>
import { useAttrs, computed, ref } from "vue";
import { createResource } from "frappe-ui";
import Autocomplete from "./Autocomplete.vue";
import { watchDebounced } from "@vueuse/core";
import { watch } from "vue";

const props = defineProps({
  doctype: {
    type: String,
    required: true,
  },
  filters: {
    type: Object,
    default: null,
  },
  modelValue: {
    type: String,
    default: "",
  },
  hideMe: {
    type: Boolean,
    default: false,
  },
  pageLength: {
    type: Number,
    default: 10,
  },
  hideClearButton: {
    type: Boolean,
    default: false,
  },
  showDescription: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits(["update:modelValue", "change"]);

const attrs = useAttrs();

const valuePropPassed = computed(() => "value" in attrs);

const value = computed({
  get: () => (valuePropPassed.value ? attrs.value : props.modelValue),
  set: (val) => {
    return (
      val?.value &&
      emit(valuePropPassed.value ? "change" : "update:modelValue", val?.value)
    );
  },
});

const autocomplete = ref(null);
const text = ref("");

watchDebounced(
  () => autocomplete.value?.query,
  (val) => {
    val = val || "";
    if (text.value === val) return;
    text.value = val;
    reload(val);
  },
  { debounce: 300, immediate: true }
);

watchDebounced(
  () => props.doctype,
  () => reload(""),
  { debounce: 300, immediate: true }
);

watch(
  () => props?.filters,
  (newVal) => {
    options.update({
      params: {
        txt: text.value,
        doctype: props.doctype,
        filters: newVal,
        page_length: props.pageLength,
      },
    });
    options.reload();
  },
  { deep: true }
);

const options = createResource({
  url: "frappe.desk.search.search_link",
  cache: [props.doctype, text.value, props.hideMe],
  method: "POST",
  params: {
    txt: text.value,
    doctype: props.doctype,
    filters: props.filters,
    page_length: props.pageLength,
  },
  transform: (data) => {
    let allData = data.map((option) => {
      return {
        value: option.value,
        label: option?.label || option.value,
        description: option?.description,
      };
    });

    if (
      !props.hideMe &&
      (props.doctype == "User" || props.doctype == "HD Agent")
    ) {
      allData.unshift({
        label: "@me",
        value: "@me",
      });
    }
    return allData;
  },
});

function reload(val) {
  if (
    options.data?.length &&
    val === options.params?.txt &&
    props.doctype === options.params?.doctype
  )
    return;

  options.update({
    params: {
      txt: val,
      doctype: props.doctype,
      filters: props.filters,
      page_length: props.pageLength,
    },
  });
  options.reload();
}

function clearValue(close) {
  emit(valuePropPassed.value ? "change" : "update:modelValue", "");
  close();
}

const labelClasses = computed(() => {
  return [
    {
      sm: "text-xs",
      md: "text-base",
    }[attrs.size || "sm"],
    "text-gray-600",
  ];
});
</script>
