<template>
  <Settings
    :title="$t('environments.environments')"
    :desc="$t('admin.settings.environments.desc')"
    docs-url="docs/usage/environments"
    :warning="$t('admin.settings.environments.warning')"
  >
    <template #titleActions>
      <Button
        v-if="selectedEnvironment"
        :text="$t('environments.show')"
        start-icon="back"
        @click="selectedEnvironment = undefined"
      />
      <Button v-else :text="$t('environments.add')" start-icon="plus" @click="showAddEnvironment" />
    </template>

    <EnvironmentList
      v-if="!selectedEnvironment"
      v-model="environments"
      :is-deleting="isDeleting"
      @edit="editEnvironment"
      @delete="deleteEnvironment"
    />

    <EnvironmentEdit
      v-else
      v-model="selectedEnvironment"
      :is-saving="isSaving"
      @save="createEnvironment"
      @cancel="selectedEnvironment = undefined"
    />
  </Settings>
</template>

<script lang="ts" setup>
import { cloneDeep } from 'lodash';
import { computed, ref } from 'vue';
import { useI18n } from 'vue-i18n';

import Button from '~/components/atomic/Button.vue';
import EnvironmentEdit from '~/components/environments/EnvironmentEdit.vue';
import EnvironmentList from '~/components/environments/EnvironmentList.vue';
import Settings from '~/components/layout/Settings.vue';
import useApiClient from '~/compositions/useApiClient';
import { useAsyncAction } from '~/compositions/useAsyncAction';
import useNotifications from '~/compositions/useNotifications';
import { usePagination } from '~/compositions/usePaginate';
import { Environment } from '~/lib/api/types';

const emptyEnvironment: Partial<Environment> = {
  name: '',
  value: '',
};

const apiClient = useApiClient();
const notifications = useNotifications();
const i18n = useI18n();

const selectedEnvironment = ref<Partial<Environment>>();
const isEditingEnvironment = computed(() => !!selectedEnvironment.value?.id);

async function loadEnvironments(page: number): Promise<Environment[] | null> {
  return apiClient.getGlobalSecretList({ page });
}

const { resetPage, data: environments } = usePagination(loadEnvironments, () => !selectedEnvironment.value);

const { doSubmit: createEnvironment, isLoading: isSaving } = useAsyncAction(async () => {
  if (!selectedEnvironment.value) {
    throw new Error("Unexpected: Can't get environment");
  }

  if (isEditingEnvironment.value) {
    await apiClient.updateGlobalSecret(selectedEnvironment.value);
  } else {
    await apiClient.createGlobalSecret(selectedEnvironment.value);
  }
  notifications.notify({
    title: i18n.t(isEditingEnvironment.value ? 'environments.saved' : 'environments.created'),
    type: 'success',
  });
  selectedEnvironment.value = undefined;
  resetPage();
});

const { doSubmit: deleteEnvironment, isLoading: isDeleting } = useAsyncAction(async (_environment: Environment) => {
  await apiClient.deleteGlobalSecret(_environment.name);
  notifications.notify({ title: i18n.t('environments.deleted'), type: 'success' });
  resetPage();
});

function editEnvironment(environment: Environment) {
  selectedEnvironment.value = cloneDeep(environment);
}

function showAddEnvironment() {
  selectedEnvironment.value = cloneDeep(emptyEnvironment);
}
</script>
