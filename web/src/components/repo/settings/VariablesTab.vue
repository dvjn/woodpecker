<template>
  <Settings :title="$t('variable.variable')" :desc="$t('variable.desc')" docs-url="docs/usage/variable">
    <template #titleActions>
      <Button
        v-if="selectedVariable"
        :text="$t('variable.show')"
        start-icon="back"
        @click="selectedVariable = undefined"
      />
      <Button v-else :text="$t('variable.add')" start-icon="plus" @click="showAddVariable" />
    </template>

    <VariableList
      v-if="!selectedVariable"
      :model-value="variables"
      :is-deleting="isDeleting"
      @edit="editVariable"
      @delete="deleteSecret"
    />

    <VariableEdit
      v-else
      v-model="selectedVariable"
      :is-saving="isSaving"
      @save="createSecret"
      @cancel="selectedVariable = undefined"
    />
  </Settings>
</template>

<script lang="ts" setup>
import { cloneDeep } from 'lodash';
import { computed, inject, Ref, ref } from 'vue';
import { useI18n } from 'vue-i18n';

import Button from '~/components/atomic/Button.vue';
import Settings from '~/components/layout/Settings.vue';
import VariableEdit from '~/components/variables/VariableEdit.vue';
import VariableList from '~/components/variables/VariableList.vue';
import useApiClient from '~/compositions/useApiClient';
import { useAsyncAction } from '~/compositions/useAsyncAction';
import useNotifications from '~/compositions/useNotifications';
import { usePagination } from '~/compositions/usePaginate';
import { Repo, Variable } from '~/lib/api/types';

const emptyVariable: Partial<Variable> = {
  name: '',
  value: '',
};

const apiClient = useApiClient();
const notifications = useNotifications();
const i18n = useI18n();

const repo = inject<Ref<Repo>>('repo');
const selectedVariable = ref<Partial<Variable>>();
const isEditingVariable = computed(() => !!selectedVariable.value?.id);

async function loadVariable(page: number, level: 'repo' | 'org' | 'global'): Promise<Variable[] | null> {
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

const { resetPage, data: _variables } = usePagination(loadVariable, () => !selectedVariable.value, {
  each: ['repo', 'org', 'global'],
  pageSize: 50,
});
const variables = computed(() => {
  const variableList: Record<string, Variable & { edit?: boolean; level: 'repo' | 'org' | 'global' }> = {};

  // eslint-disable-next-line no-restricted-syntax
  for (const level of ['repo', 'org', 'global']) {
    // eslint-disable-next-line no-restricted-syntax
    for (const variable of _variables.value) {
      if (
        ((level === 'repo' && variable.repo_id !== 0 && variable.org_id === 0) ||
          (level === 'org' && variable.repo_id === 0 && variable.org_id !== 0) ||
          (level === 'global' && variable.repo_id === 0 && variable.org_id === 0)) &&
        !variableList[variable.name]
      ) {
        variableList[variable.name] = { ...variable, edit: variable.repo_id !== 0, level };
      }
    }
  }

  const levelsOrder = {
    global: 0,
    org: 1,
    repo: 2,
  };

  return Object.values(variableList)
    .toSorted((a, b) => a.name.localeCompare(b.name))
    .toSorted((a, b) => levelsOrder[b.level] - levelsOrder[a.level]);
});

const { doSubmit: createSecret, isLoading: isSaving } = useAsyncAction(async () => {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  if (!selectedVariable.value) {
    throw new Error("Unexpected: Can't get variable");
  }

  if (isEditingVariable.value) {
    await apiClient.updateSecret(repo.value.id, selectedVariable.value);
  } else {
    await apiClient.createSecret(repo.value.id, selectedVariable.value);
  }
  notifications.notify({
    title: i18n.t(isEditingVariable.value ? 'variable.saved' : 'variable.created'),
    type: 'success',
  });
  selectedVariable.value = undefined;
  await resetPage();
});

const { doSubmit: deleteSecret, isLoading: isDeleting } = useAsyncAction(async (_variable: Variable) => {
  if (!repo?.value) {
    throw new Error("Unexpected: Can't load repo");
  }

  await apiClient.deleteSecret(repo.value.id, _variable.name);
  notifications.notify({ title: i18n.t('variable.deleted'), type: 'success' });
  await resetPage();
});

function editVariable(variable: Variable) {
  selectedVariable.value = cloneDeep(variable);
}

function showAddVariable() {
  selectedVariable.value = cloneDeep(emptyVariable);
}
</script>
