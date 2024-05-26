<template>
  <div class="space-y-4 text-wp-text-100">
    <ListItem
      v-for="environment in environments"
      :key="environment.id"
      class="items-center !bg-wp-background-200 !dark:bg-wp-background-100"
    >
      <span>{{ environment.name }}</span>
      <Badge
        v-if="environment.edit === false"
        class="ml-2"
        :label="environment.org_id === 0 ? $t('global_level_environment') : $t('org_level_environment')"
      />
      <div class="ml-auto space-x-2 <md:hidden">
        <Badge v-for="event in environment.events" :key="event" :label="event" />
      </div>
      <template v-if="environment.edit !== false">
        <IconButton
          icon="edit"
          class="ml-2 <md:ml-auto w-8 h-8"
          :title="$t('environments.edit')"
          @click="editEnvironment(environment)"
        />
        <IconButton
          icon="trash"
          class="ml-2 w-8 h-8 hover:text-wp-control-error-100"
          :is-loading="isDeleting"
          :title="$t('environments.delete')"
          @click="deleteEnvironment(environment)"
        />
      </template>
    </ListItem>

    <div v-if="environments?.length === 0" class="ml-2">{{ $t('environments.none') }}</div>
  </div>
</template>

<script lang="ts" setup>
import { toRef } from 'vue';
import { useI18n } from 'vue-i18n';

import Badge from '~/components/atomic/Badge.vue';
import IconButton from '~/components/atomic/IconButton.vue';
import ListItem from '~/components/atomic/ListItem.vue';
import { Environment } from '~/lib/api/types';

const props = defineProps<{
  modelValue: (Environment & { edit?: boolean })[];
  isDeleting: boolean;
}>();

const emit = defineEmits<{
  (event: 'edit', environment: Environment): void;
  (event: 'delete', environment: Environment): void;
}>();

const i18n = useI18n();

const environments = toRef(props, 'modelValue');

function editEnvironment(environment: Environment) {
  emit('edit', environment);
}

function deleteEnvironment(environment: Environment) {
  // TODO: use proper dialog
  // eslint-disable-next-line no-alert, no-restricted-globals
  if (!confirm(i18n.t('repo.settings.environments.delete_confirm'))) {
    return;
  }
  emit('delete', environment);
}
</script>
