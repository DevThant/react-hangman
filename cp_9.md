Thank you for providing the `useSync.ts` composable and related store. Based on this, we can implement the WebSocket solution to communicate sync progress across different browser instances.

### Step 1: Update `useSync.ts` to emit progress events via WebSocket

We'll modify `useSync.ts` to send progress events via WebSocket when the sync status changes.

#### useSync.ts

```typescript
import { eventService, EventType } from '@/services/event.js';
import { toastService, ToastType } from '@/services/toast.js';
import { useProductsStore } from '@/stores/products.js';
import { useAppStore } from '@/stores/app.js';
import { DialogNames } from '@/typings/dialog.js';
import { SyncDialogType } from '@/typings/sync.js';
import { onBeforeUnmount, Ref, ref, watch } from 'vue';
import {
  MXSynchronizationProgressSubscription,
  MXSyncProgressStatus
} from '@ebitoolmx/gateway-types';
import { useAuthService } from '@/auth/index.js';
import { websocketService, Topic } from '@/services/webSocket'; // Import WebSocket service

export function useSync(
  isSyncEnabled: Ref<boolean>,
  syncStatusSubscription: Ref<MXSynchronizationProgressSubscription | null>
) {
  const appStore = useAppStore();
  const productsStore = useProductsStore();
  const { userDetails } = useAuthService();

  const syncProcessingId = ref<symbol | null>(null);
  const taskId = ref<symbol>(Symbol(productsStore.activeProductId));

  watch(
    () => productsStore.activeProductId,
    (newId, oldId) => {
      if (newId !== oldId) {
        appStore.removeBlockingTask(taskId.value);
        taskId.value = Symbol(newId);
      }
    }
  );

  watch(
    [syncStatusSubscription, () => isSyncEnabled.value],
    ([newStatus], [oldStatus]) => {
      if (!newStatus || !isSyncEnabled.value) {
        appStore.removeBlockingTask(taskId.value);
        return;
      }

      const newSyncStatus = newStatus.synchronizationProgress?.syncProgressStatus;
      const olsSyncStatus = oldStatus?.synchronizationProgress?.syncProgressStatus;

      const syncedByUser = newStatus.synchronizationProgress?.syncedBy === userDetails.value?.email;
      const syncProcessing = newSyncStatus === MXSyncProgressStatus.Synchronizing;

      const syncInitialising = newSyncStatus === MXSyncProgressStatus.Initializing;

      const syncSuccessful =
        (newSyncStatus === MXSyncProgressStatus.Successful &&
          olsSyncStatus === MXSyncProgressStatus.Synchronizing) ||
        (newSyncStatus === MXSyncProgressStatus.Successful &&
          olsSyncStatus === MXSyncProgressStatus.Initializing);

      const syncUnsuccessful =
        newSyncStatus === MXSyncProgressStatus.Unsuccessful &&
        olsSyncStatus === MXSyncProgressStatus.Synchronizing;

      const syncInitialisationFailed =
        newSyncStatus === MXSyncProgressStatus.InitializationFailed &&
        olsSyncStatus === MXSyncProgressStatus.Initializing;

      if (syncProcessing) {
        appStore.addBlockingTask(taskId.value);
      } else {
        appStore.removeBlockingTask(taskId.value);
        if (syncProcessingId.value) {
          toastService.dismiss(syncProcessingId.value);
          syncProcessingId.value = null;
        }
      }

      // Send sync progress to WebSocket
      websocketService.send(Topic.ProductUpdate, {
        type: 'SYNC_PROGRESS',
        progress: newStatus.synchronizationProgress,
      });

      if (syncedByUser) {
        if (syncInitialising) {
          eventService.emit(EventType.OpenDialog, {
            dialogName: DialogNames.Sync,
            options: {
              props: {
                dialogType: SyncDialogType.Initialising,
                status: newStatus.synchronizationProgress
              },
              modal: true
            }
          });
        } else if (syncProcessing)
          eventService.emit(EventType.OpenDialog, {
            dialogName: DialogNames.Sync,
            options: {
              props: {
                dialogType: SyncDialogType.Processing,
                status: newStatus.synchronizationProgress
              },
              modal: true
            }
          });
        else if (syncSuccessful)
          eventService.emit(EventType.OpenDialog, {
            dialogName: DialogNames.Sync,
            options: {
              props: {
                dialogType: SyncDialogType.Successful,
                status: newStatus.synchronizationProgress
              },
              modal: true
            }
          });
        else if (syncInitialisationFailed) {
          eventService.emit(EventType.OpenDialog, {
            dialogName: DialogNames.Sync,
            options: {
              props: {
                dialogType: SyncDialogType.InitialisingFailed,
                status: newStatus.synchronizationProgress
              },
              modal: true
            }
          });
        } else if (syncUnsuccessful)
          eventService.emit(EventType.OpenDialog, {
            dialogName: DialogNames.Sync,
            options: {
              props: {
                dialogType: SyncDialogType.Unsuccessful,
                status: newStatus.synchronizationProgress
              },
              modal: true
            }
          });
      } else if ((syncProcessing || syncInitialising) && !syncProcessingId.value) {
        syncProcessingId.value = toastService.display(ToastType.Spinner, {
          type: 'default',
          title: 'sync.toaster.requested',
          titleVars: {
            syncedBy: newStatus.synchronizationProgress?.syncedBy
          },
          dismissable: true
        });
      } else if (syncSuccessful) {
        toastService.display(ToastType.Simple, {
          type: 'default',
          title: 'sync.toaster.successful'
        });
      } else if (syncUnsuccessful || syncInitialisationFailed) {
        toastService.display(ToastType.Simple, {
          type: 'default',
          title: 'sync.toaster.unsuccessful'
        });
      }
    },
    { immediate: true }
  );

  onBeforeUnmount(() => {
    if (syncProcessingId.value) toastService.dismiss(syncProcessingId.value);
    appStore.removeBlockingTask(taskId.value);
  });
}
```

### Step 2: Update `Spinner.vue` to listen to WebSocket events

We'll update `Spinner.vue` to listen for WebSocket events and update the progress message accordingly.

#### spinner.vue

```vue
<template>
  <div class="item default">
    <div class="header">
      <span class="title semi-bold">{{ combinedMessage }}</span>
      <app-icon-button
        v-if="dismissable"
        :title="t('common.dismiss')"
        :name="'times'"
        class="icon-button dismiss"
        @click="dismiss"
      />
    </div>
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted, computed } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { websocketService, Topic } from '@/services/webSocket'; // Import WebSocket service

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false
  }
);

const emit = defineEmits<{ dismiss: [id: symbol] }>();

const { t, te } = useI18n();

const dismiss = () => emit('dismiss', props.id);

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const titleText = computed(() => (te(props.title) ? t(props.title, props.titleVars) : props.title));
const progressMessage = ref('');

const updateProgressMessage = (status) => {
  console.log('Received sync progress event:', status);
  progressMessage.value = `${status.syncedModels.length}/${status.allModels.length} models are done.`;
};

const combinedMessage = computed(() => {
  return `${titleText.value}\n${progressMessage.value}`;
});

onMounted(() => {
  console.log('Spinner mounted and listening for events');
  toastService.on(ToastEvent.Dismiss, onDismiss);

  // Subscribe to WebSocket events
  const subscriptionId = websocketService.subscribe(Topic.ProductUpdate, (message) => {
    if (message.type === 'SYNC_PROGRESS') {
      updateProgressMessage(message.progress);
    }
  });

  // Cleanup subscription on unmount
  onBeforeUnmount(() => {
    console.log('Spinner before unmount and removing event listeners');
    toastService.off(ToastEvent.Dismiss, onDismiss);
    websocketService.unsubscribe(subscriptionId);
  });
});
</script>

<style scoped>
.icon-button {
  background-color: transparent;
  border-width: 0;
  padding: 0;
  height: 16px;
  width: 16px;
  cursor: pointer;
}

.header {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  width: 100%;
}

.header .title {
  flex-grow: 1;
  padding-right: var(--base-spacing-0);
}

.header .dismiss {
  fill: var(--toaster-dismiss-icon);
  flex-shrink: 0;
  height: 16px;
  width: 16px;
  font-size

: 16px;
  display: inline-block;
}

.progress-info {
  margin-top: 10px;
}

.spinner {
  margin: 15px 0 15px;
}
</style>
```

### Step 3: Ensure WebSocket server is correctly handling and broadcasting events

Make sure your WebSocket server is correctly handling and broadcasting the sync progress events to all connected clients.

With these steps, you should be able to communicate sync progress updates via WebSocket, allowing the `Spinner.vue` component to receive real-time updates even across different browser instances.
