<template>
  <div class="absolute bottom-0 z-50 w-full p-4">
    <ui-card class="h-full w-full" padding="p-0">
      <div class="flex items-center p-4">
        <ui-input
          v-model="state.query"
          :placeholder="$t('common.search')"
          autofocus
          autocomplete="off"
          prepend-icon="riSearch2Line"
        />
        <div class="grow" />
        <ui-button icon @click="$emit('close')">
          <v-remixicon name="riCloseLine" />
        </ui-button>
      </div>
      <div
        class="scroll mx-4 flex space-x-4 overflow-x-auto pb-4"
        style="min-height: 95px"
      >
        <p
          v-if="packageStore.packages.length === 0"
          class="w-full py-8 text-center"
        >
          {{ t('message.noData') }}
        </p>
        <div
          v-for="item in items"
          :key="item.id"
          draggable="true"
          class="hoverable relative flex shrink-0 cursor-move flex-col rounded-lg border-2 transition"
          style="width: 288px; height: 125px"
          @dragstart="
            $event.dataTransfer.setData('savedBlocks', JSON.stringify(item))
          "
        >
          <div class="flex flex-1 items-start p-4">
            <div
              v-if="item.icon"
              :class="{ 'mr-2': item.icon.startsWith('http') }"
              class="w-8 shrink-0"
            >
              <img
                v-if="item.icon.startsWith('http')"
                :src="item.icon"
                width="38"
                height="38"
                class="rounded-lg"
              />
              <v-remixicon
                v-else
                :name="item.icon || 'mdiPackageVariantClosed'"
              />
            </div>
            <div class="flex-1 overflow-hidden">
              <p class="text-overflow font-semibold leading-tight">
                {{ item.name }}
              </p>
              <p
                class="line-clamp leading-tight text-gray-600 dark:text-gray-200"
              >
                {{ item.description }}
              </p>
            </div>
          </div>
          <div
            class="flex items-center space-x-3 px-4 pb-4 text-gray-600 dark:text-gray-200"
          >
            <span v-if="item.author" class="text-overflow">
              By {{ item.author }}
            </span>
            <div class="grow" />
            <a
              v-if="item.isExternal"
              :href="`https://extension.automa.site/packages/${item.id}`"
              target="_blank"
              title="Open package page"
            >
              <v-remixicon name="riExternalLinkLine" size="18" />
            </a>
            <ui-popover style="height: 18px">
              <template #trigger>
                <v-remixicon
                  size="18"
                  class="cursor-pointer"
                  name="riMore2Line"
                />
              </template>
              <ui-list>
                <ui-list-item
                  v-close-popover
                  class="cursor-pointer"
                  @click="updatePackages(item)"
                >
                  Update packages
                  <v-remixicon
                    v-tooltip="
                      'Update the current package inside the workflow.'
                    "
                    class="ml-2 -mr-1"
                    name="riInformationLine"
                    size="20"
                  />
                </ui-list-item>
                <ui-list-item
                  v-close-popover
                  class="cursor-pointer text-red-400 dark:text-red-500"
                  @click="deleteItem(item)"
                >
                  {{ t('common.delete') }}
                </ui-list-item>
              </ui-list>
            </ui-popover>
          </div>
        </div>
      </div>
    </ui-card>
  </div>
</template>
<script setup>
import { useDialog } from '@/composable/dialog';
import { usePackageStore } from '@/stores/package';
import { computed, inject, reactive } from 'vue';
import { useI18n } from 'vue-i18n';

defineEmits(['close']);

const { t } = useI18n();
const dialog = useDialog();
const packageStore = usePackageStore();

const state = reactive({
  query: '',
});

const editor = inject('workflow-editor');

const sortedItems = computed(() =>
  packageStore.packages.slice().sort((a, b) => b.createdAt - a.createdAt)
);
const items = computed(() => {
  const query = state.query.toLocaleLowerCase();

  return sortedItems.value.filter((item) =>
    item.name.toLocaleLowerCase().includes(query)
  );
});

function deleteItem({ id, name }) {
  dialog.confirm({
    title: 'Delete package',
    body: `Are you sure want to delete "${name}" package?`,
    okText: 'Delete',
    okVariant: 'danger',
    onConfirm: () => {
      packageStore.delete(id);
    },
  });
}
function removeConnections({ id, type, oldEdges, newEdges }) {
  const removedEdges = [];
  oldEdges.forEach((edge) => {
    const isNotDeleted = newEdges.find((item) => item.id === edge.id);
    if (isNotDeleted) return;

    const handleType = type.slice(0, -1);

    removedEdges.push(`${id}-${handleType}-${edge.id}`);
  });

  const edgesToRemove = editor.value.getEdges.value.filter(
    ({ sourceHandle, targetHandle }) => {
      if (type === 'outputs') {
        return removedEdges.includes(sourceHandle);
      }

      return removedEdges.includes(targetHandle);
    }
  );

  editor.value.removeEdges(edgesToRemove);
}
function updatePackages(item) {
  const packageNodes = editor.value.getNodes.value.filter(
    (node) => node.data.id === item.id
  );
  if (packageNodes.length === 0) return;

  packageNodes.forEach((node) => {
    removeConnections({
      id: node.id,
      type: 'inputs',
      newEdges: item.inputs,
      oldEdges: node.data.inputs,
    });
    removeConnections({
      id: node.id,
      type: 'outputs',
      newEdges: item.outputs,
      oldEdges: node.data.outputs,
    });

    node.data = { ...item };
  });
}
</script>
