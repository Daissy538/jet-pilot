<script setup lang="ts">
import { injectStrict } from "@/lib/utils";
import { V1Job } from "@kubernetes/client-node";
import { Kubernetes } from "@/services/Kubernetes";
import { ref, h } from "vue";
import { useToast, ToastAction } from "@/components/ui/toast";

import { KubeContextStateKey } from "@/providers/KubeContextProvider";
const { context, namespace, kubeConfig } = injectStrict(KubeContextStateKey);

import DataTable from "@/components/ui/VirtualDataTable.vue";
import { columns } from "@/components/tables/jobs";
import { useDataRefresher } from "@/composables/refresher";

const { toast } = useToast();
const jobs = ref<V1Job[]>([]);

import { RowAction, getDefaultActions } from "@/components/tables/types";
import { PanelProviderAddTabKey } from "@/providers/PanelProvider";
const addTab = injectStrict(PanelProviderAddTabKey);

import { DialogProviderSpawnDialogKey } from "@/providers/DialogProvider";
const spawnDialog = injectStrict(DialogProviderSpawnDialogKey);

const rowActions: RowAction<V1Job>[] = [
  ...getDefaultActions<V1Job>(
    addTab,
    spawnDialog,
    context.value,
    kubeConfig.value
  ),
];

import { useRoute } from "vue-router";
const route = useRoute();
const rowClasses = (row: any) => {
  if (route.query.uid) {
    return row.metadata.uid === route.query.uid
      ? "animate-pulse-highlight-once"
      : "";
  }

  return "";
};

async function getJobs(refresh: boolean = false) {
  if (!refresh) {
    jobs.value = [];
  }

  Kubernetes.getJobs(
    context.value,
    namespace.value === "all" ? "" : namespace.value
  )
    .then((results: V1Job[]) => {
      jobs.value = results;
    })
    .catch((error) => {
      toast({
        title: "An error occured",
        description: error.message,
        variant: "destructive",
        action: h(
          ToastAction,
          { altText: "Retry", onClick: () => startRefreshing() },
          { default: () => "Retry" }
        ),
      });
      stopRefreshing();
    });
}

const { startRefreshing, stopRefreshing } = useDataRefresher(getJobs, 1000, [
  context,
  namespace,
]);

const create = () => {
  addTab(
    `create_` + Math.random().toString(36).substring(7),
    `New Job`,
    defineAsyncComponent(() => import("@/views/ObjectEditor.vue")),
    {
      context: context,
      namespace: namespace.value === "all" ? "" : namespace,
      kubeConfig: kubeConfig,
      create: true,
      type: "job",
      useKubeCtl: false,
    },
    "edit"
  );
};
</script>

<template>
  <DataTable
    :data="jobs"
    :columns="columns"
    :allow-filter="true"
    :sticky-headers="true"
    :row-actions="rowActions"
    :row-classes="rowClasses"
  >
    <template #action-buttons>
      <button
        class="transition-all ml-2 hover:opacity-100 opacity-50 z-50 rounded-full w-9 h-9 flex items-center justify-center bg-primary text-white text-lg"
        @click="create"
      >
        +
      </button>
    </template>
  </DataTable>
</template>
