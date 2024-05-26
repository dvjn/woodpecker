<template>
  <div v-if="innerValue" class="space-y-4">
    <form @submit.prevent="save">
      <InputField v-slot="{ id }" :label="$t('environments.name')">
        <TextField
          :id="id"
          v-model="innerValue.name"
          :placeholder="$t('environments.name')"
          required
          :disabled="isEditingEnvironment"
        />
      </InputField>

      <InputField v-slot="{ id }" :label="$t('environments.value')">
        <TextField
          :id="id"
          v-model="innerValue.value"
          :placeholder="$t('environments.value')"
          :lines="5"
          :required="!isEditingEnvironment"
        />
      </InputField>

      <div class="flex gap-2">
        <Button type="button" color="gray" :text="$t('cancel')" @click="$emit('cancel')" />
        <Button
          type="submit"
          color="green"
          :is-loading="isSaving"
          :text="isEditingEnvironment ? $t('environments.save') : $t('environments.add')"
        />
      </div>
    </form>
  </div>
</template>

<script lang="ts" setup>
import { computed, toRef } from 'vue';

import Button from '~/components/atomic/Button.vue';
import InputField from '~/components/form/InputField.vue';
import TextField from '~/components/form/TextField.vue';
import { Environment } from '~/lib/api/types';

const props = defineProps<{
  modelValue: Partial<Environment>;
  isSaving: boolean;
}>();

const emit = defineEmits<{
  (event: 'update:modelValue', value: Partial<Environment> | undefined): void;
  (event: 'save', value: Partial<Environment>): void;
  (event: 'cancel'): void;
}>();

const modelValue = toRef(props, 'modelValue');
const innerValue = computed({
  get: () => modelValue.value,
  set: (value) => {
    emit('update:modelValue', value);
  },
});
const isEditingEnvironment = computed(() => !!innerValue.value?.id);

function save() {
  if (!innerValue.value) {
    return;
  }

  emit('save', innerValue.value);
}
</script>
