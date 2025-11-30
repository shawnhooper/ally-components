<script setup lang="ts">
import { computed, watch, inject, onUnmounted, ref } from 'vue';
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
const optionRefs = ref<Record<string | number, HTMLInputElement | null>>({});

const describedBy = computed(() => {
  const ids = [
    props.ariaDescribedby,
    props.helptext ? helpTextId.value : undefined,
    isInvalid.value ? errorTextId.value : undefined,
  ].filter(Boolean);
  const uniqueIds = Array.from(new Set(ids)).filter(id => id !== labelId.value);
  return uniqueIds.length ? uniqueIds.join(' ') : undefined;
});

const isOptionDisabled = (option: RadioOption) => props.disabled || option.disabled;

const focusOption = (value: string | number) => {
  const el = optionRefs.value[value];
  if (el) el.focus();
};

const onKeydown = (event: KeyboardEvent, option: RadioOption) => {
  if (event.key !== 'ArrowRight' && event.key !== 'ArrowDown' && event.key !== 'ArrowLeft' && event.key !== 'ArrowUp') {
    return;
  }
  event.preventDefault();
  const enabledOptions = props.options.filter(opt => !isOptionDisabled(opt));
  const currentIndex = enabledOptions.findIndex(opt => opt.value === option.value);
  if (currentIndex === -1 || enabledOptions.length === 0) return;

  const nextIndex = (event.key === 'ArrowRight' || event.key === 'ArrowDown')
    ? (currentIndex + 1) % enabledOptions.length
    : (currentIndex - 1 + enabledOptions.length) % enabledOptions.length;

  const nextOption = enabledOptions[nextIndex];
  emit('update:modelValue', nextOption.value);
  focusOption(nextOption.value);
};

const setOptionRef = (value: string | number, el: unknown) => {
  optionRefs.value[value] = el as HTMLInputElement | null;
};
</script>

<template>
  <fieldset :id="id" class="form-group" :aria-invalid="isInvalid ? 'true' : undefined">
    <legend :id="labelId" class="weight-400 text-black font-size-14">
      {{ label }}
      <span v-if="required" aria-hidden="true" class="text-danger ms-1">*</span>
    </legend>
    <small v-if="helptext" :id="helpTextId" class="form-text text-muted">
      {{ helptext }}
    </small>
    <div class="multi-radio-group">
      <div v-for="option in options" :key="option.value" class="form-check custom-radio">
        <input
          :id="`${id}-${option.value}`"
          class="custom-control-input"
          type="radio"
          :name="id"
          :value="option.value"
          :checked="selectedValue === option.value"
          :disabled="isOptionDisabled(option)"
          :required="required"
          :aria-required="required ? 'true' : undefined"
          :aria-invalid="isInvalid ? 'true' : undefined"
          :aria-labelledby="option.label ? `${labelId} ${id}-${option.value}-label` : labelId"
          :aria-describedby="describedBy"
          @change="emit('update:modelValue', option.value)"
          @keydown="onKeydown($event, option)"
          @blur="$emit('blur', $event)"
          :tabindex="(disabled || option.disabled)
            ? -1
            : (selectedValue === option.value || (!selectedValue && options[0].value === option.value) ? 0 : -1)"
          :ref="el => setOptionRef(option.value, el)"
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
.error-text {
  display: block;
  color: #B22222;
  margin-top: 0.25rem;
  font-size: 0.875rem;
}

.error-text.reserve-space {
  visibility: hidden;
}
.custom-radio .custom-control-input:focus + .custom-control-label::before {
    box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25);
    border-color: #80bdff;
}
.custom-radio .custom-control-input.focus + .custom-control-label::before {
    box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25);
    border-color: #80bdff;
}
</style>
