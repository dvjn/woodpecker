<template>
  <Settings :title="$t('environments.environments')" :desc="$t('environments.desc')" docs-url="docs/usage/environments">
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
      :model-value="environments"
      :is-deleting="isDeleting"
      @edit="editEnvironment"
      @delete="deleteSecret"
    />

    <EnvironmentEdit
      v-else
      v-model="selectedEnvironment"
      :is-saving="isSaving"
      @save="createSecret"
      @cancel="selectedEnvironment = undefined"
    />
  </Settings>
</template>

<script lang="ts" setup>
import { cloneDeep } from 'lodash';
import { computed, inject, Ref, ref } from 'vue';
import { useI18n } from 'vue-i18n';

import Button from '~/components/atomic/Button.vue';
import EnvironmentEdit from '~/components/environments/EnvironmentEdit.vue';
import EnvironmentList from '~/components/environments/EnvironmentList.vue';
import Settings from '~/components/layout/Settings.vue';
import useApiClient from '~/compositions/useApiClient';
import { useAsyncAction } from '~/compositions/useAsyncAction';
import useNotifications from '~/compositions/useNotifications';
import { usePagination } from '~/compositions/usePaginate';
import { Environment, Repo } from '~/lib/api/types';

const emptyEnvironment: Partial<Environment> = {
  name: '',
  value: '',
};

const apiClient = useApiClient();
const notifications = useNotifications();
const i18n = useI18n();

const repo = inject<Ref<Repo>>('repo');
const selectedEnvironment = ref<Partial<Environment>>();
const isEditingEnvironment = computed(() => !!selectedEnvironment.value?.id);

async function loadEnvironment(page: number, level: 'repo' | 'org' | 'global'): Promise<Environment[] | null> {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  switch (level) {
    case 'repo':
      return apiClient.getSecretList(repo.value.id, { page });
    case 'org':
      return apiClient.getOrgSecretList(repo.value.org_id, { page });
    case 'global':
      return apiClient.getGlobalSecretList({ page });
    default:
      throw new Error(`Unexpected level: ${level}`);
  }
}

const { resetPage, data: _environments } = usePagination(loadEnvironment, () => !selectedEnvironment.value, {
  each: ['repo', 'org', 'global'],
  pageSize: 50,
});
const environments = computed(() => {
  const environmentsList: Record<string, Environment & { edit?: boolean; level: 'repo' | 'org' | 'global' }> = {};

  // eslint-disable-next-line no-restricted-syntax
  for (const level of ['repo', 'org', 'global']) {
    // eslint-disable-next-line no-restricted-syntax
    for (const environment of _environments.value) {
      if (
        ((level === 'repo' && environment.repo_id !== 0 && environment.org_id === 0) ||
          (level === 'org' && environment.repo_id === 0 && environment.org_id !== 0) ||
          (level === 'global' && environment.repo_id === 0 && environment.org_id === 0)) &&
        !environmentsList[environment.name]
      ) {
        environmentsList[environment.name] = { ...environment, edit: environment.repo_id !== 0, level };
      }
    }
  }

  const levelsOrder = {
    global: 0,
    org: 1,
    repo: 2,
  };

  return Object.values(environmentsList)
    .toSorted((a, b) => a.name.localeCompare(b.name))
    .toSorted((a, b) => levelsOrder[b.level] - levelsOrder[a.level]);
});

const { doSubmit: createSecret, isLoading: isSaving } = useAsyncAction(async () => {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  if (!selectedEnvironment.value) {
    throw new Error("Unexpected: Can't get environment");
  }

  if (isEditingEnvironment.value) {
    await apiClient.updateSecret(repo.value.id, selectedEnvironment.value);
  } else {
    await apiClient.createSecret(repo.value.id, selectedEnvironment.value);
  }
  notifications.notify({
    title: i18n.t(isEditingEnvironment.value ? 'environments.saved' : 'environments.created'),
    type: 'success',
  });
  selectedEnvironment.value = undefined;
  await resetPage();
});

const { doSubmit: deleteSecret, isLoading: isDeleting } = useAsyncAction(async (_environment: Environment) => {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  await apiClient.deleteSecret(repo.value.id, _environment.name);
  notifications.notify({ title: i18n.t('environments.deleted'), type: 'success' });
  await resetPage();
});

function editEnvironment(environment: Environment) {
  selectedEnvironment.value = cloneDeep(environment);
}

function showAddEnvironment() {
  selectedEnvironment.value = cloneDeep(emptyEnvironment);
}
</script>
