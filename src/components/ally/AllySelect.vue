<script setup lang="ts">
import { computed, inject, watch, onUnmounted, useSlots } from 'vue';
import { AllyFormKey } from './allyFormKeys';

interface SelectOption {
  value: string | number;
  label: string;
  disabled?: boolean;
}

const props = withDefaults(defineProps<{
  id: string;
  label: string;
  modelValue: string | number;
  options: SelectOption[];
  required?: boolean;
  placeholder?: string;
  ariaDescribedby?: string;
  errorMessage?: string;
  reserveErrorSpace?: boolean;
}>(), {
  required: false,
  placeholder: 'Select an option',
  ariaDescribedby: undefined,
  errorMessage: '',
  reserveErrorSpace: true,
});

const emit = defineEmits(['update:modelValue', 'blur']);

// Inject form context
const allyForm = inject(AllyFormKey, null);

// Watch for error message changes and report to parent
watch(() => props.errorMessage, (newMessage, oldMessage) => {
  if (allyForm) {
    allyForm.updateErrorState(props.id, newMessage || '');
  }
}, { immediate: true });

// Watch for ID changes and clean up old errors
watch(() => props.id, (newId, oldId) => {
  if (allyForm && oldId && props.errorMessage) {
    allyForm.clearErrorState(oldId);
  }
  if (allyForm) {
    allyForm.updateErrorState(newId, props.errorMessage || '');
  }
});

// Clean up on unmount
onUnmounted(() => {
  if (allyForm) {
    allyForm.clearErrorState(props.id);
  }
});

// Computed property to handle select event and emit update
const value = computed({
  get: () => props.modelValue,
  set: (newValue) => {
    emit('update:modelValue', newValue);
  }
});

// Generate unique IDs for descriptive elements
const helpTextId = computed(() => `${props.id}-help`);
const errorTextId = computed(() => `${props.id}-error`);
const labelId = computed(() => `${props.id}-label`);

// Compute invalid state based on errorMessage prop
const isInvalid = computed(() => !!props.errorMessage);

const slots = useSlots();
const describedBy = computed(() => {
  const ids = [
    props.ariaDescribedby,
    slots.helptext ? helpTextId.value : undefined,
    isInvalid.value ? errorTextId.value : undefined,
  ].filter(Boolean);
  const uniqueIds = Array.from(new Set(ids)).filter(id => id !== labelId.value);
  return uniqueIds.length ? uniqueIds.join(' ') : undefined;
});
</script>

<template>
  <div class="form-group" :class="{ 'ally-has-error': isInvalid }">
    <label v-if="label" :id="labelId" :for="id" class="form-label">
      {{ label }}
      <span v-if="required" aria-hidden="true" class="text-danger ms-1">*</span>
    </label>

    <div class="input-wrapper position-relative">
      <select
        :id="id"
        v-model="value"
        :class="['form-select', { 'is-invalid': isInvalid }]"
        :aria-required="required"
        :aria-invalid="isInvalid ? 'true' : undefined"
        :aria-labelledby="label ? labelId : undefined"
        :aria-describedby="describedBy"
        @blur="$emit('blur', $event)"
      >
        <option value="" disabled :aria-label="placeholder">{{ placeholder }}</option>
        <option
          v-for="option in options"
          :key="option.value"
          :value="option.value"
          :disabled="option.disabled"
          :aria-label="option.label"
        >
          {{ option.label }}
        </option>
      </select>
    </div>

    <!-- Help Text -->
    <small v-if="$slots.helptext" :id="helpTextId" class="form-text text-muted">
      <slot name="helptext" />
    </small>

    <!-- Error Message Area -->
    <div v-if="isInvalid || reserveErrorSpace"
         :id="errorTextId"
         class="error-text"
         :class="{ 'reserve-space': reserveErrorSpace && !isInvalid }"
         >
      {{ errorMessage }}
    </div>
  </div>
</template>

<style scoped>
/* Use custom error color for the required asterisk */
.form-label span[aria-hidden="true"] {
  color: #B22222;
  margin-left: 0.25rem;
}

/* Override Bootstrap invalid state colors */
.form-select.is-invalid {
  border-color: #B22222;
}

.form-select.is-invalid:focus {
  border-color: #B22222;
  box-shadow: 0 0 0 0.2rem rgba(178, 34, 34, 0.25);
}

/* Base styles for the error message div */
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
