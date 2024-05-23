Given the frustration you're experiencing and the need to directly reflect sync status updates in the spinner, another approach is to use a global event bus or directly use the same event mechanism that `sync.vue` uses to update its status. This can help ensure that both components (the sync dialog and the spinner toast) are synchronized in their updates.

### Step-by-Step Solution Using Global Event Bus

1. **Use a Global Event Bus**: This ensures that sync status updates are broadcasted globally and any component can listen to these updates.
2. **Update Both Components**: Ensure that both `sync.vue` and `spinner.vue` listen to the same global events.

Let's implement this:

### 1. Create a Global Event Bus

Create an event bus that will handle the sync progress events.

#### eventBus.js

```javascript
import mitt from 'mitt';
export const eventBus = mitt();
```

### 2. Update `sync.vue` to Emit Events on the Event Bus

Update `sync.vue` to emit sync progress events to the global event bus.

#### sync.vue

```vue
<template>
  <app-dialog-container title-key="sync.modal.title" data-testid="dialog-sync">
    <template #content>
      <app-dialog-process-content-layout
        v-if="isSyncProcessingDialog"
        :header="t(header)"
        :description-line1="message ? t(message) : message"
      >
        <template #extra-content>
          <ul class="sync-progress lead dialog-content-message">
            <li
              v-for="(model, index) of status?.allModels"
              :key="index"
              :class="{
                syncing: status?.currentModelName === model,
                synced: status?.syncedModels.includes(model)
              }"
            >
              <app-icon :name="progressIcon(model)" />
              {{ model }}
            </li>
          </ul>
        </template>
      </app-dialog-process-content-layout>

      <app-dialog-icon-content-layout
        v-else
        :icon="icon"
        :icon-color="iconColor"
        :header="t(header)"
        :message-line1="message ? t(message) : message"
      >
        <template #extra-content>
          <div v-if="isSyncUnsuccessfulDialog || isSyncFailedDialog" class="unsuccessful">
            <div class="unsuccessful-message">
              <p v-if="isSyncUnsuccessfulDialog" class="lead dialog-content-message">
                {{ t('sync.modal.message.unsuccessful') }}
              </p>
              <p class="lead dialog-content-message">{{ t('sync.modal.message.contact') }}</p>
              <a
                :href="t('common.link')"
                rel="noopener noreferrer"
                target="_blank"
                class="lead dialog-content-message"
              >
                {{ t('common.helpSupport') }}
              </a>
              <p class="lead dialog-content-message">.</p>
            </div>

            <div v-if="status?.currentModelName && isSyncUnsuccessfulDialog" class="failed-model">
              <app-icon name="sync_error" />
              <p class="lead dialog-content-message">{{ status.currentModelName }}</p>
            </div>
          </div>
          <div v-else-if="isSyncSuccessfulDialog">
            <ul class="sync-progress lead dialog-content-message">
              <li v-for="(model, index) of status?.syncedModels" :key="index" class="synced">
                <app-icon name="sync_success" />
                {{ model }}
              </li>
            </ul>
          </div>
        </template>
      </app-dialog-icon-content-layout>
    </template>

    <template #button-row>
      <app-button
        data-testid="reject-sync"
        :disabled="appStore.hasBlockingTasks"
        :class="{ primary: !isSyncRequestDialog, secondary: isSyncRequestDialog }"
        @click="emit('close')"
        >{{ t('common.close') }}</app-button
      >
      <app-button
        v-if="isSyncRequestDialog"
        data-testid="accept-sync"
        class="primary"
        :disabled="startDisabled"
        @click="start"
        >{{ t('common.start') }}</app-button
      >
    </template>
  </app-dialog-container>
</template>
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
import { eventBus } from '@/services/eventBus'; // Import the event bus

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

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });
}

watch(
  () =>

 props.status,
  newStatus => {
    if (newStatus) {
      console.log('Sync progress event:', newStatus);
      eventBus.emit('syncProgress', newStatus); // Emit sync progress event on the global event bus
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

<style scoped>
.sync-progress {
  color: var(--font-disabled-color);
  fill: var(--font-disabled-color);

  li {
    display: flex;
    align-items: center;

    .icon {
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
    fill: var(--success);
  }
}

.unsuccessful {
  display: flex;
  flex-direction: column;
  align-items: center;

  .unsuccessful-message {
    display: flex;

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }
}
</style>
```

### 3. Update `spinner.vue` to Listen to Global Events

Update `spinner.vue` to listen to the sync progress events from the global event bus.

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
import { eventBus } from '@/services/eventBus'; // Import the global event bus

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
  eventBus.on('syncProgress', updateProgressMessage); // Listen for sync progress events
});

onBeforeUnmount(() => {
  console.log('Spinner before unmount and removing event listeners');
  toastService.off(ToastEvent.Dismiss, onDismiss);
  eventBus.off('syncProgress', updateProgressMessage); // Stop listening for sync progress events
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

### Summary:

1. **Global Event Bus**: Created a global event bus using `mitt`.
2. **sync.vue**: Updated to emit sync progress events on the global event bus.
3. **spinner.vue**: Updated to listen to sync progress events from the global event bus.

By using a global event bus, you ensure that all components that need to react to the sync progress updates can do so without being directly dependent on each other. This should help ensure that the `spinner.vue` component receives the sync status updates and reflects them correctly.
