<template>
  <Panel>
    <div class="flex flex-row border-b mb-4 pb-4 items-center dark:border-wp-background-100">
      <div class="ml-2">
        <h1 class="text-xl text-wp-text-100">{{ $t('user.settings.variables.variables') }}</h1>
        <p class="text-sm text-wp-text-alt-100">
          {{ $t('user.settings.variables.desc') }}
          <DocsLink :topic="$t('user.settings.variables.variables')" url="docs/usage/variables" />
        </p>
      </div>
      <Button
        v-if="selectedVariable"
        class="ml-auto"
        :text="$t('variables.show')"
        start-icon="back"
        @click="selectedVariable = undefined"
      />
      <Button v-else class="ml-auto" :text="$t('variables.add')" start-icon="plus" @click="showAddVariable" />
    </div>

    <VariableList
      v-if="!selectedVariable"
      v-model="variables"
      :is-deleting="isDeleting"
      @edit="editVariable"
      @delete="deleteVariable"
    />

    <VariableEdit
      v-else
      v-model="selectedVariable"
      :is-saving="isSaving"
      @save="createVariable"
      @cancel="selectedVariable = undefined"
    />
  </Panel>
</template>

<script lang="ts" setup>
import { cloneDeep } from 'lodash';
import { computed, ref } from 'vue';
import { useI18n } from 'vue-i18n';

import Button from '~/components/atomic/Button.vue';
import DocsLink from '~/components/atomic/DocsLink.vue';
import Panel from '~/components/layout/Panel.vue';
import VariableEdit from '~/components/variables/VariableEdit.vue';
import VariableList from '~/components/variables/VariableList.vue';
import useApiClient from '~/compositions/useApiClient';
import { useAsyncAction } from '~/compositions/useAsyncAction';
import useAuthentication from '~/compositions/useAuthentication';
import useNotifications from '~/compositions/useNotifications';
import { usePagination } from '~/compositions/usePaginate';
import { Variable } from '~/lib/api/types';

const emptyVariable: Partial<Variable> = {
  name: '',
  value: '',
};

const apiClient = useApiClient();
const notifications = useNotifications();
const i18n = useI18n();

const { user } = useAuthentication();
if (!user) {
  throw new Error('Unexpected: Unauthenticated');
}
const selectedVariable = ref<Partial<Variable>>();
const isEditingVariable = computed(() => !!selectedVariable.value?.id);

async function loadVariables(page: number): Promise<Variable[] | null> {
  if (!user) {
    throw new Error('Unexpected: Unauthenticated');
  }

  return apiClient.getOrgSecretList(user.org_id, { page });
}

const { resetPage, data: variables } = usePagination(loadVariables, () => !selectedVariable.value);

const { doSubmit: createVariable, isLoading: isSaving } = useAsyncAction(async () => {
  if (!selectedVariable.value) {
    throw new Error("Unexpected: Can't get variable");
  }

  if (isEditingVariable.value) {
    await apiClient.updateOrgSecret(user.org_id, selectedVariable.value);
  } else {
    await apiClient.createOrgSecret(user.org_id, selectedVariable.value);
  }
  notifications.notify({
    title: i18n.t(isEditingVariable.value ? 'user.settings.variables.saved' : 'user.settings.variables.created'),
    type: 'success',
  });
  selectedVariable.value = undefined;
  resetPage();
});

const { doSubmit: deleteVariable, isLoading: isDeleting } = useAsyncAction(async (_variable: Variable) => {
  await apiClient.deleteOrgSecret(user.org_id, _variable.name);
  notifications.notify({ title: i18n.t('user.settings.variables.deleted'), type: 'success' });
  resetPage();
});

function editVariable(variable: Variable) {
  selectedVariable.value = cloneDeep(variable);
}

function showAddVariable() {
  selectedVariable.value = cloneDeep(emptyVariable);
}
</script>
