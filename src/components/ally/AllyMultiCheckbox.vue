<script setup lang="ts">
import { computed, watch, inject, onUnmounted } from 'vue';
import AllyCheckbox from './AllyCheckbox.vue';
import { AllyFormKey } from './allyFormKeys';

interface CheckboxOption {
  label: string;
  value: string | number;
  disabled?: boolean;
}

const props = withDefaults(defineProps<{
  id: string;
  label: string;
  options: CheckboxOption[];
  modelValue?: Array<string | number>;
  required?: boolean;
  errorMessage?: string;
  helptext?: string;
  disabled?: boolean;
  reserveErrorSpace?: boolean;
  minSelectedItems?: number;
  maxSelectedItems?: number;
  ariaDescribedby?: string;
}>(), {
  required: false,
  errorMessage: '',
  helptext: '',
  disabled: false,
  reserveErrorSpace: true,
  modelValue: () => [],
  minSelectedItems: 0,
  maxSelectedItems: undefined, // will default to options.length in setup
  ariaDescribedby: undefined,
});

const emit = defineEmits(['update:modelValue', 'blur']);

const allyForm = inject(AllyFormKey, null);

// Default maxSelectedItems to options.length if not provided
const maxSelected = computed(() =>
  props.maxSelectedItems !== undefined ? props.maxSelectedItems : props.options.length
);

const isInvalid = computed(() => !!validationError.value || !!props.errorMessage);

const selectedValues = computed({
  get: () => props.modelValue ?? [],
  set: (val) => emit('update:modelValue', val),
});

const validationError = computed(() => {
  if (props.disabled) return '';
  const count = selectedValues.value.length;
  if (count < props.minSelectedItems) {
    return `Please select at least ${props.minSelectedItems} option${props.minSelectedItems > 1 ? 's' : ''}.`;
  }
  if (count > maxSelected.value) {
    return `Please select no more than ${maxSelected.value} option${maxSelected.value > 1 ? 's' : ''}.`;
  }
  return '';
});

// Error reporting to AllyForm
watch([() => props.errorMessage, validationError], ([newMessage, newValidation]) => {
  if (allyForm) {
    allyForm.updateErrorState(props.id, newMessage || newValidation || '');
  }
}, { immediate: true });

watch(() => props.id, (newId, oldId) => {
  if (allyForm) {
    if (oldId && (props.errorMessage || validationError.value)) {
      allyForm.clearErrorState(oldId);
    }
    allyForm.updateErrorState(newId, props.errorMessage || validationError.value || '');
  }
});

onUnmounted(() => {
  if (allyForm) {
    allyForm.clearErrorState(props.id);
  }
});

function toggleValue(optionValue: string | number) {
  const current = new Set(selectedValues.value);
  if (current.has(optionValue)) {
    current.delete(optionValue);
  } else {
    current.add(optionValue);
  }
  selectedValues.value = Array.from(current);
}

const helpTextId = computed(() => `${props.id}-help`);
const errorTextId = computed(() => `${props.id}-error`);
const labelId = computed(() => `${props.id}-label`);

const describedBy = computed(() => {
  const ids = [
    props.ariaDescribedby,
    props.helptext ? helpTextId.value : undefined,
    isInvalid.value ? errorTextId.value : undefined,
  ].filter(Boolean);
  const uniqueIds = Array.from(new Set(ids)).filter(id => id !== labelId.value);
  return uniqueIds.length ? uniqueIds.join(' ') : undefined;
});
</script>

<template>
  <fieldset
    :id="id"
    class="form-group"
    :aria-invalid="isInvalid ? 'true' : undefined"
    :aria-describedby="describedBy"
  >
    <legend :id="labelId" class="weight-400 text-black font-size-14">
      {{ label }}
      <span v-if="required" aria-hidden="true" class="text-danger ms-1">*</span>
    </legend>
    <div class="multi-checkbox-group">
      <AllyCheckbox
        v-for="option in options"
        :key="option.value"
        :id="`${id}-${option.value}`"
        :label="option.label"
        :ariaLabelledby="option.label ? `${labelId} ${id}-${option.value}-label` : labelId"
        :modelValue="selectedValues.includes(option.value)"
        :disabled="disabled || option.disabled"
        :required="false"
        @update:modelValue="val => toggleValue(option.value)"
        @blur="$emit('blur', $event)"
      />
    </div>
    <small v-if="helptext" :id="helpTextId" class="form-text text-muted">
      {{ helptext }}
    </small>
    <div v-if="isInvalid || reserveErrorSpace"
         :id="errorTextId"
         class="error-text"
         :class="{ 'reserve-space': reserveErrorSpace && !isInvalid }"
         >
      {{ errorMessage || validationError }}
    </div>
  </fieldset>
</template>

<style scoped>
.multi-checkbox-group {
  display: flex;
  flex-direction: column;
  gap: 0;
}
.error-text {
  display: block;
  color: #B22222;
  margin-top: 0.25rem;
  font-size: 0.875rem;
}

.error-text.reserve-space {
  visibility: hidden;
}
</style>
