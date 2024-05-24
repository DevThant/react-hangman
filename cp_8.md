In your current setup, the `Toaster` component relies on the `ToastService` to display toast messages. When someone triggers a sync, the `Sync.vue` component needs to notify the `ToastService` to display a spinner toast, and then the `Spinner.vue` should update based on the sync progress events.

### Steps to achieve the desired functionality:

1. **Trigger Spinner Toast Display in `Sync.vue`**: When a sync is triggered, `Sync.vue` should notify `ToastService` to display a spinner toast.

2. **Send Sync Progress Events via WebSocket**: Use the `WebSocketService` to broadcast sync progress events.

3. **Update Spinner Toast with Sync Progress Events**: The `Spinner.vue` component should listen for sync progress events from the `WebSocketService`.

Let's implement these steps:

### 1. Trigger Spinner Toast Display in `Sync.vue`

Update `Sync.vue` to notify `ToastService` to display a spinner toast when a sync is triggered.

#### sync.vue

```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { PickupPaths, useI18n } from 'vue-i18n';
import { exhaustiveTypeCheck } from '@ebitoolmx/predicates';
import AppIcon from '@/components/common/icon/Icon.vue';
import AppButton from '@/components/common/formElements/button/Button.vue';
import AppDialogContainer from '@/components/common/dialog/layouts/DialogContainer.vue';
import AppDialogIconContentLayout, { IconColor } from '@/components/common/dialog/layouts/DialogIconContentLayout.vue';
import AppDialogProcessContentLayout from '@/components/common/dialog/layouts/DialogProcessContentLayout.vue';
import { SyncDialogType } from '@/typings/sync.js';
import { DialogNames } from '@/typings/dialog.js';
import { useAppStore } from '@/stores/app.js';
import { useProductsStore } from '@/stores/products.js';
import { useConnectionsStore } from '@/stores/connections.js';
import { useEditorStore } from '@/stores/editor.js';
import { eventService, EventType } from '@/services/event.js';
import { LocaleMessage } from '@/locale/en.js';
import { MXSyncProgressEvent, MXSyncProgressStatus } from '@ebitoolmx/gateway-types';
import { useAuthService } from '@/auth/index.js';
import { websocketService, Topic } from '@/services/webSocket.js';
import { toastService, ToastType } from '@/services/toast.js';

defineOptions({ name: 'Sync' });

const props = defineProps<{
  dialogType: SyncDialogType;
  status?: MXSyncProgressEvent;
}>();

const emit = defineEmits(['close']);
const appStore = useAppStore();
const { userDetails } = useAuthService();
const connectionsStore = useConnectionsStore();
const productsStore = useProductsStore();
const editorStore = useEditorStore();
const { t } = useI18n();

const header = computed<PickupPaths<LocaleMessage>>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return 'sync.modal.header.requested';
    case SyncDialogType.Initialising:
      return 'sync.modal.header.initialising';
    case SyncDialogType.Processing:
      return 'sync.modal.header.processing';
    case SyncDialogType.Successful:
      return 'sync.modal.header.successful';
    case SyncDialogType.InitialisingFailed:
      return 'sync.modal.header.initialisingFailed';
    case SyncDialogType.Unsuccessful:
      return 'sync.modal.header.unsuccessful';
    case SyncDialogType.Offline:
      return 'sync.modal.header.offline';
    default:
      return exhaustiveTypeCheck(props.dialogType);
  }
});

const message = computed<PickupPaths<LocaleMessage> | null>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return 'sync.modal.message.requested';
    case SyncDialogType.Processing:
      return 'sync.modal.message.progress';
    case SyncDialogType.Offline:
      return 'sync.modal.message.offline';
    case SyncDialogType.Initialising:
      return null;
    case SyncDialogType.Successful:
      return null;
    case SyncDialogType.InitialisingFailed:
      return null;
    case SyncDialogType.Unsuccessful:
      return null;
    default:
      return exhaustiveTypeCheck(props.dialogType);
  }
});

const progressIcon = (model: string) => {
  if (props.status?.syncedModels.includes(model)) {
    return 'sync_success';
  }
  if (props.status?.currentModelName === model) {
    return 'sync_in_progress';
  }
  return 'sync_no_progress';
};

const icon = computed<string>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return 'sync';
    case SyncDialogType.Successful:
      return 'sync_success';
    case SyncDialogType.Unsuccessful:
      return 'warning';
    case SyncDialogType.InitialisingFailed:
      return 'warning';
    case SyncDialogType.Offline:
      return 'cloud-offline';
    default:
      return '';
  }
});

const iconColor = computed<IconColor>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return 'text';
    case SyncDialogType.Successful:
      return 'success';
    case SyncDialogType.Unsuccessful:
      return 'danger';
    case SyncDialogType.InitialisingFailed:
      return 'danger';
    case SyncDialogType.Offline:
      return 'warning';
    default:
      return 'text';
  }
});

const isSyncRequestDialog = computed<boolean>(() => props.dialogType === SyncDialogType.Requested);

const isSyncProcessingDialog = computed<boolean>(
  () =>
    props.dialogType === SyncDialogType.Processing ||
    props.dialogType === SyncDialogType.Initialising
);

const isSyncSuccessfulDialog = computed<boolean>(
  () => props.dialogType === SyncDialogType.Successful
);

const isSyncUnsuccessfulDialog = computed<boolean>(
  () => props.dialogType === SyncDialogType.Unsuccessful
);

const isSyncFailedDialog = computed<boolean>(
  () => props.dialogType === SyncDialogType.InitialisingFailed
);

const startDisabled = computed<boolean>(
  () =>
    appStore.hasBlockingTasks ||
    props.status?.syncProgressStatus === MXSyncProgressStatus.Synchronizing
);

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  const toastId = Symbol('syncToast');
  toastService.displaySpinner(toastId, t('sync.toaster.requested', { syncedBy: email }));

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });
}

watch(
  () => props.status,
  newStatus => {
    if (newStatus) {
      console.log('Sync progress event:', newStatus);
      websocketService.send(Topic.ProductUpdate, newStatus); // Send the sync progress event using WebSocket
    }
  },
  { immediate: true }
);

watch(
  () => connectionsStore.isOffline,
  nowOffline => {
    if (nowOffline)
      eventService.emit(EventType.OpenDialog, {
        dialogName: DialogNames.Sync,
        options: {
          props: {
            dialogType: SyncDialogType.Offline
          },
          modal: true
        }
      });
  }
);
</script>
```

### 2. Update `Spinner.vue` to Listen to WebSocket Messages

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
import { websocketService, Topic } from '@/services/webSocket.js';

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
  if (status) {
    console.log('Received sync progress event:', status);
    progressMessage.value = `${status.syncedModels.length}/${status.allModels.length} models are done.`;
  }
};

const combinedMessage = computed(() => {
  return `${titleText.value}\n${progressMessage.value}`;
});

onMounted(() => {
  console.log('Spinner mounted and listening for events');
  toastService.on(ToastEvent.Dismiss, onDismiss);

  const subscriptionId = websocketService.subscribe(Topic.ProductUpdate, (message) => {
    updateProgressMessage(message.message);
  });
});

onBeforeUnmount(() => {
  console.log('Spinner before unmount and removing event listeners');
  toastService.off(ToastEvent.Dismiss, onDismiss);
  websocketService.unsubscribe(subscriptionId);
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
  font-size: 16px;
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

### Summary

1. **WebSocket Integration**: 
   - `Sync.vue` sends progress updates via WebSocket.
   - `Spinner.vue` listens for these updates via WebSocket.

2. **Real-time Updates**: All connected clients receive real-time sync progress updates.

This approach ensures that the sync progress status is communicated in real-time to all relevant clients, leveraging the existing WebSocket infrastructure.