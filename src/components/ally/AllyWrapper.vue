<script setup lang="ts">
import { computed, inject, watch, onUnmounted, useSlots } from 'vue';
import { AllyFormKey } from './allyFormKeys'; // Import the injection key

// Define component props
const props = withDefaults(defineProps<{
  id: string;
  label: string;
  modelValue: string | number; // Support v-model
  required?: boolean;
  ariaDescribedby?: string; // For linking external descriptions
  errorMessage?: string; // Added error message prop
  reserveErrorSpace?: boolean; // Added prop back here
  autocomplete?: string; // Added autocomplete prop
}>(), {
  required: false,
  ariaDescribedby: undefined,
  errorMessage: '', // Default error message is empty
  reserveErrorSpace: true, // Default to true
  autocomplete: undefined, // Default autocomplete
});

// Define emits for v-model support
const emit = defineEmits(['update:modelValue', 'blur']);

// --- Inject context from AllyForm (only error reporting) ---
const allyForm = inject(AllyFormKey, null); // Inject with null default

// Watch for error message changes and report to parent
watch(() => props.errorMessage, (newMessage, oldMessage) => {
  if (allyForm) {
    allyForm.updateErrorState(props.id, newMessage || '');
  }
}, { immediate: true });

// Watch for ID changes and clean up old errors
watch(() => props.id, (newId, oldId) => {
  if (allyForm && oldId && props.errorMessage) {
    // If the ID changes *while* there's an error, clear the error associated with the old ID
    allyForm.clearErrorState(oldId);
  }
  // Report the current error state under the new ID
  if (allyForm) {
      allyForm.updateErrorState(newId, props.errorMessage || '');
  }
});

// --- Clean up on unmount ---
onUnmounted(() => {
  if (allyForm) { // Explicit check for injected context
    allyForm.clearErrorState(props.id);
  }
});

// Computed property to handle input event and emit update
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

// Build aria-describedby while avoiding duplicate IDs and ignoring the label ID
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
    <!-- New wrapper for input and counter -->
    <div class="input-wrapper position-relative">
      <slot 
        name="control" 
        :id="id"
        :isInvalid="isInvalid"
        :describedBy="describedBy"
        :labelId="labelId"
        :required="required"
        :autocomplete="autocomplete"
      />
    </div>
    <!-- Help Text (outside input-wrapper) -->
    <small v-if="$slots.helptext" :id="helpTextId" class="form-text text-muted">
      <slot name="helptext" />
    </small>

    <!-- Error Message Area (Simplified) -->
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
  color: #B22222; /* Custom Error Color */
  margin-left: 0.25rem;
}

/* Override Bootstrap invalid state colors */
.form-control.is-invalid {
  border-color: #B22222; /* Custom Error Color */
}

.form-control.is-invalid:focus {
  border-color: #B22222;
  /* Replicate Bootstrap 4 focus shadow with the custom color */
  box-shadow: 0 0 0 0.2rem rgba(178, 34, 34, 0.25); /* rgba(178, 34, 34) is #B22222 */
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
