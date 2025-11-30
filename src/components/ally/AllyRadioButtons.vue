<script setup lang="ts">
import { computed, watch, inject, onUnmounted } from 'vue';
import { AllyFormKey } from './allyFormKeys';

interface RadioOption {
  label: string;
  value: string | number;
  disabled?: boolean;
}

const props = withDefaults(defineProps<{
  id: string;
  label: string;
  options: RadioOption[];
  modelValue?: string | number;
  required?: boolean;
  errorMessage?: string;
  helptext?: string;
  disabled?: boolean;
  reserveErrorSpace?: boolean;
  ariaDescribedby?: string;
}>(), {
  required: false,
  errorMessage: '',
  helptext: '',
  disabled: false,
  reserveErrorSpace: true,
  modelValue: '',
  ariaDescribedby: undefined,
});

const emit = defineEmits(['update:modelValue', 'blur']);

const allyForm = inject(AllyFormKey, null);

// Error reporting to AllyForm
watch(() => props.errorMessage, (newMessage) => {
  if (allyForm) {
    allyForm.updateErrorState(props.id, newMessage || '');
  }
}, { immediate: true });

watch(() => props.id, (newId, oldId) => {
  if (allyForm) {
    if (oldId && props.errorMessage) {
      allyForm.clearErrorState(oldId);
    }
    allyForm.updateErrorState(newId, props.errorMessage || '');
  }
});

onUnmounted(() => {
  if (allyForm) {
    allyForm.clearErrorState(props.id);
  }
});

const isInvalid = computed(() => !!props.errorMessage);

const selectedValue = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val),
});

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
  <fieldset :id="id" class="form-group" :aria-invalid="isInvalid ? 'true' : undefined">
    <legend :id="labelId" class="multi-radio-legend">
      {{ label }}
      <span v-if="required" aria-hidden="true" class="text-danger ms-1">*</span>
    </legend>
    <div class="multi-radio-group">
      <div v-for="option in options" :key="option.value" class="form-check">
        <input
          :id="`${id}-${option.value}`"
          class="form-check-input"
          type="radio"
          :name="id"
          :value="option.value"
          :checked="selectedValue === option.value"
          :disabled="disabled || option.disabled"
          :required="required"
          :aria-required="required ? 'true' : undefined"
          :aria-invalid="isInvalid ? 'true' : undefined"
          :aria-labelledby="option.label ? `${labelId} ${id}-${option.value}-label` : labelId"
          :aria-describedby="describedBy"
          @change="emit('update:modelValue', option.value)"
          @blur="$emit('blur', $event)"
          :tabindex="(disabled || option.disabled)
            ? -1
            : (selectedValue === option.value || (!selectedValue && options[0].value === option.value) ? 0 : -1)"
        />
        <label
          v-if="option.label"
          class="form-check-label"
          :id="`${id}-${option.value}-label`"
          :for="`${id}-${option.value}`"
        >
          {{ option.label }}
        </label>
      </div>
    </div>
    <small v-if="helptext" :id="helpTextId" class="form-text text-muted">
      {{ helptext }}
    </small>
    <div v-if="isInvalid || reserveErrorSpace"
         :id="errorTextId"
         class="error-text"
         :class="{ 'reserve-space': reserveErrorSpace && !isInvalid }"
         >
      {{ errorMessage }}
    </div>
  </fieldset>
</template>

<style scoped>
.multi-radio-group {
  display: flex;
  flex-direction: column;
  gap: 0;
}
.multi-radio-legend {
  font-size: 1rem;
  font-weight: 500;
  margin-bottom: 0.5rem;
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
