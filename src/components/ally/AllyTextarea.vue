<script setup lang="ts">
import { computed, inject, watch, onUnmounted, useSlots } from 'vue';
import { AllyFormKey } from './allyFormKeys';

const props = withDefaults(defineProps<{
  id: string;
  label: string;
  modelValue: string;
  required?: boolean;
  placeholder?: string;
  ariaDescribedby?: string;
  errorMessage?: string;
  maxlength?: number;
  showCounter?: boolean;
  reserveErrorSpace?: boolean;
  rows?: number;
}>(), {
  required: false,
  placeholder: '',
  ariaDescribedby: undefined,
  errorMessage: '',
  maxlength: undefined,
  showCounter: false,
  reserveErrorSpace: true,
  rows: 3
});

const emit = defineEmits(['update:modelValue', 'blur']);

const allyForm = inject(AllyFormKey, null);

watch(() => props.errorMessage, (newMessage, oldMessage) => {
  if (allyForm) {
    allyForm.updateErrorState(props.id, newMessage || '');
  }
}, { immediate: true });

watch(() => props.id, (newId, oldId) => {
  if (allyForm && oldId && props.errorMessage) {
    allyForm.clearErrorState(oldId);
  }
  if (allyForm) {
    allyForm.updateErrorState(newId, props.errorMessage || '');
  }
});

onUnmounted(() => {
  if (allyForm) {
    allyForm.clearErrorState(props.id);
  }
});

const value = computed({
  get: () => props.modelValue,
  set: (newValue) => {
    emit('update:modelValue', newValue);
  }
});

const helpTextId = computed(() => `${props.id}-help`);
const errorTextId = computed(() => `${props.id}-error`);
const counterId = computed(() => `${props.id}-counter`);
const labelId = computed(() => `${props.id}-label`);

const isInvalid = computed(() => !!props.errorMessage);
const currentLength = computed(() => String(props.modelValue || '').length);
const shouldShowCounter = computed(() => props.showCounter && props.maxlength !== undefined);

const slots = useSlots();
const describedBy = computed(() => {
  const ids = [
    props.ariaDescribedby,
    slots.helptext ? helpTextId.value : undefined,
    isInvalid.value ? errorTextId.value : undefined,
    shouldShowCounter.value ? counterId.value : undefined,
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
    <div class="textarea-wrapper position-relative">
      <textarea
        :id="id"
        v-model="value"
        :class="['form-control', { 'is-invalid': isInvalid, 'has-counter': shouldShowCounter }]"
        :placeholder="placeholder"
        :maxlength="maxlength"
        :rows="rows"
        :aria-required="required"
        :aria-invalid="isInvalid ? 'true' : undefined"
        :aria-labelledby="label ? labelId : undefined"
        :aria-describedby="describedBy"
        @blur="$emit('blur', $event)"
      ></textarea>
      <small v-if="shouldShowCounter" :id="counterId" class="form-text text-muted char-counter">
        {{ currentLength }} / {{ maxlength }}
      </small>
    </div>
    <small v-if="$slots.helptext" :id="helpTextId" class="form-text text-muted">
      <slot name="helptext" />
    </small>

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
.textarea-wrapper {
  position: relative;
}

.form-control.has-counter {
  padding-bottom: 2rem;
}

.char-counter {
  position: absolute;
  bottom: 0.25rem;
  right: 0.5rem;
  font-size: 0.75em;
  pointer-events: none;
}

.form-label span[aria-hidden="true"] {
  color: #B22222;
  margin-left: 0.25rem;
}

.form-control.is-invalid {
  border-color: #B22222;
}

.form-control.is-invalid:focus {
  border-color: #B22222;
  box-shadow: 0 0 0 0.2rem rgba(178, 34, 34, 0.25);
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
