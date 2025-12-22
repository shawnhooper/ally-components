<script setup lang="ts">
import { computed, inject, watch, onUnmounted } from 'vue';
import { AllyFormKey } from './allyFormKeys'; // Import the injection key

// Define valid variants (adjust as needed)
type CheckboxVariant = 'primary' | 'secondary' | 'success' | 'danger' | 'warning' | 'info' | 'dark' | 'light';

// Define component props
const props = withDefaults(defineProps<{
  id: string;
  label: string;
  modelValue: boolean; // v-model should be boolean for single checkbox
  required?: boolean;
  disabled?: boolean; // Added disabled prop
  ariaDescribedby?: string; // For linking external descriptions
  errorMessage?: string; // For validation error
  reserveErrorSpace?: boolean; // Add reserveErrorSpace prop
  variant?: CheckboxVariant; // Added variant prop
}>(), {
  required: false,
  disabled: false, // Default disabled to false
  ariaDescribedby: undefined,
  errorMessage: '', // Default error message is empty
  reserveErrorSpace: true, // Default to true
  variant: 'primary', // Default to primary
});

// Define emits for v-model support
const emit = defineEmits(['update:modelValue', 'blur']); // Include blur if needed

// --- Inject context from AllyForm (error reporting) ---
const allyForm = inject(AllyFormKey, null);

// Watch for error message changes and report to parent AllyForm
watch(() => props.errorMessage, (newMessage) => {
  if (allyForm) {
    allyForm.updateErrorState(props.id, newMessage || '');
  }
}, { immediate: true });

// Watch for ID changes and update error state in AllyForm
watch(() => props.id, (newId, oldId) => {
  if (allyForm) {
    if (oldId && props.errorMessage) {
      allyForm.clearErrorState(oldId); // Clear error associated with the old ID
    }
    allyForm.updateErrorState(newId, props.errorMessage || ''); // Report error under new ID
  }
});

// --- Clean up error state on unmount ---
onUnmounted(() => {
  if (allyForm) {
    allyForm.clearErrorState(props.id);
  }
});

// Computed property to handle v-model for checkbox
const checked = computed({
  get: () => props.modelValue,
  set: (newValue) => {
    emit('update:modelValue', newValue);
  }
});

// Generate unique IDs for descriptive elements
const helpTextId = computed(() => `${props.id}-help`);
const errorTextId = computed(() => `${props.id}-error`);

// Compute invalid state based on errorMessage prop
const isInvalid = computed(() => !!props.errorMessage);

// Computed class for the input based on variant and state
const inputClass = computed(() => [
  'form-check-input',
  { 'is-invalid': isInvalid.value && !props.disabled }
]);

const inputAriaLabel = computed(() => props.label || undefined);
</script>

<template>
  <div class="form-group" :class="{ 'ally-has-error': isInvalid, 'ally-disabled': disabled }">
    <div class="custom-checkbox form-check">
      <input
        :id="id"
        type="checkbox"
        class="form-check-input custom-control-input"
        :class="{ 'is-invalid': isInvalid }"
        v-model="checked"
        :required="required"
        :disabled="disabled"
        tabindex="0"
        :aria-label="inputAriaLabel"
        :aria-required="required ? 'true' : undefined"
        :aria-invalid="isInvalid ? 'true' : undefined"
        :aria-describedby="[
          props.ariaDescribedby,
          $slots.helptext ? helpTextId : undefined,
          isInvalid ? errorTextId : undefined,
        ].filter(Boolean).join(' ') || undefined"
        @blur="$emit('blur', $event)"
      />
      <label
        v-if="label"
        class="custom-control-label form-check-label"
        :for="id"
        aria-hidden="true"
      >
        {{ label }}
        <span v-if="required && !disabled" aria-hidden="true" class="text-danger ms-1">*</span>
      </label>
    </div>

    <!-- Help Text -->
    <small v-if="$slots.helptext" :id="helpTextId" class="form-text text-muted d-block mt-1">
      <slot name="helptext" />
    </small>

    <!-- Error Message Area -->
    <div v-if="(isInvalid || reserveErrorSpace) && !disabled"
         :id="errorTextId"
         class="error-text d-block"
         :class="{ 'reserve-space': reserveErrorSpace && !isInvalid }"
         >
      {{ errorMessage }}
    </div>

  </div>
</template>

<style scoped>
/* Use custom error color for the required asterisk on the label */
.custom-control-label span[aria-hidden="true"] {
  color: #B22222; /* Match Textbox Error Color */
  margin-left: 0.25rem;
}

/* Ensure error message and help text have spacing */
.error-text, .form-text {
  display: block;
  color: #B22222;
  font-size: 0.875rem;
}

.error-text.reserve-space {
  visibility: hidden;
}

/* Apply error styles to the input's sibling label for BS4 */
.custom-control-input.is-invalid ~ .custom-control-label {
  color: #B22222; /* Example: Make label text red */
}
.custom-control-input.is-invalid ~ .custom-control-label::before {
  border-color: #B22222; /* Make the box border red */
}

/* Add default focus style */

/* Style the label text color when the parent group indicates an error */
.ally-has-error .custom-control-label {
   /* Optionally change label color on error */
   /* color: #B22222; */
}

/* Disabled Styles for BS4 custom control */
.ally-disabled .custom-control-label {
  opacity: 0.65;
  cursor: not-allowed;
}

/* BS4 handles input opacity/cursor via the :disabled attribute and label styling */
.custom-control-input:disabled ~ .custom-control-label {
  opacity: 0.65;
  cursor: not-allowed;
}

/* Ensure proper spacing for the form group */
.form-group {
  margin-bottom: 1rem; /* Adjust as needed */
}

.custom-checkbox {
  display: flex;
  align-items: flex-start;
  gap: 0.5rem;
  padding-left: 0;
}

.custom-checkbox .form-check-input {
  margin-top: 0.2rem;
  margin-left: 0;
  float: none;
  flex-shrink: 0;
}

.custom-checkbox .custom-control-label {
  margin-bottom: 0;
  line-height: 1.4;
  flex: 1;
}

</style>
<style>
.custom-control-input:focus ~ .custom-control-label::before {
  border-color: #80bdff; /* Default BS4 focus border color */
  box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25); /* Default BS4 focus shadow */
}

/* Add focus styles matching BS4 invalid state */
.custom-control-input.is-invalid:focus ~ .custom-control-label::before {
  border-color: #B22222;
  box-shadow: 0 0 0 0.2rem rgba(178, 34, 34, 0.25);
}
</style>
