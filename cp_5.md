To ensure that the `syncStore` is initialized before `spinner.vue` attempts to access it, you can implement a check or wait mechanism to delay the mount of the spinner until the store is ready. One way to achieve this is to add a check in the spinner component to wait until the store status is initialized before proceeding with rendering or watching the status.

Here are the steps to implement this:

1. **Add a readiness state to syncStore**: You can add a boolean state to track if the store is ready.

### syncStore.ts
```typescript
import { defineStore } from 'pinia';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

export const useSyncStore = defineStore('sync', {
  state: () => ({
    status: null as MXSyncProgressEvent | null,
    isReady: false // Add this state to track readiness
  }),
  actions: {
    setStatus(newStatus: MXSyncProgressEvent) {
      this.status = newStatus;
      this.isReady = true; // Set readiness to true when status is updated
    }
  }
});
```

2. **Modify spinner.vue to wait for store readiness**: Implement a mechanism to wait until the store is ready before performing operations.

### Spinner.vue
```vue
<template>
  <div v-if="syncStore.isReady" class="item default">
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
  <div v-else class="item default">
    <!-- Optionally, you can show a loading spinner or some placeholder while waiting -->
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed, onMounted, onBeforeUnmount } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { useSyncStore } from '@/stores/syncStore';

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
const syncStore = useSyncStore();
const dismiss = () => emit('dismiss', props.id);

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const titleText = computed(() => te(props.title) ? t(props.title, props.titleVars) : props.title);
const progressMessage = ref('');

watch(
  () => syncStore.status,
  (newStatus) => {
    if (newStatus) {
      console.log('Updating progress message with new store status:', newStatus);
      progressMessage.value = `${newStatus.syncedModels.length}/${newStatus.allModels.length} models are done.`;
    } else {
      console.log('syncStore.status is undefined or null');
    }
  },
  { immediate: true }
);

const combinedMessage = computed(() => {
  return `${titleText.value}\n${progressMessage.value}`;
});

onMounted(() => {
  console.log('Spinner mounted and listening for events');
  toastService.on(ToastEvent.Dismiss, onDismiss);
  console.log('syncStore status on mount:', syncStore.status);
});

onBeforeUnmount(() => {
  console.log('Spinner before unmount and removing event listeners');
  toastService.off(ToastEvent.Dismiss, onDismiss);
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

### Explanation:
1. **syncStore.ts**: Added an `isReady` state to track whether the store status has been initialized.
2. **Spinner.vue**: Added a conditional check in the template to render the content only if `syncStore.isReady` is `true`. If not, it shows a placeholder spinner. 

This approach ensures that the spinner component waits until the `syncStore` is initialized and ready before attempting to access its status.


--------------

Given that the current approach using an event bus isn't working as expected, we can consider another approach to ensure `Spinner.vue` receives the necessary sync progress updates. Let's use a shared store via Pinia to manage the sync state. This will ensure that any component that needs to access the sync state, including `Spinner.vue`, can do so reactively.

### Step-by-Step Solution

1. **Create a Pinia Store for Sync State:**
   - Create a new Pinia store to manage the sync state.

2. **Update `Sync.vue` to Update the Store:**
   - Ensure that `Sync.vue` updates the Pinia store with the sync progress.

3. **Update `Spinner.vue` to Read from the Store:**
   - Modify `Spinner.vue` to read the sync state from the Pinia store.

### 1. Create a Pinia Store for Sync State

First, ensure you have Pinia installed in your project. If not, you can install it using:

```bash
npm install pinia
```

Then, create a Pinia store for the sync state. Let's create a new file `stores/syncStore.ts`.

```typescript
import { defineStore } from 'pinia';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

export const useSyncStore = defineStore('sync', {
  state: () => ({
    status: null as MXSyncProgressEvent | null
  }),
  actions: {
    setStatus(status: MXSyncProgressEvent) {
      this.status = status;
    }
  }
});
```

### 2. Update `Sync.vue` to Update the Store

Modify `Sync.vue` to update the Pinia store whenever the sync status changes.

```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { PickupPaths, useI18n } from 'vue-i18n';

import { exhaustiveTypeCheck } from '@ebitoolmx/predicates';

import AppIcon from '@/components/common/icon/Icon.vue';
import AppButton from '@/components/common/formElements/button/Button.vue';
import AppDialogContainer from '@/components/common/dialog/layouts/DialogContainer.vue';
import AppDialogIconContentLayout, {
  IconColor
} from '@/components/common/dialog/layouts/DialogIconContentLayout.vue';
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
import { useSyncStore } from '@/stores/syncStore'; // Import the sync store

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
const syncStore = useSyncStore(); // Use the sync store
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
    case SyncDialogType.Unsuccessful':
      return 'danger';
    case SyncDialogType.InitialisingFailed:
      return 'danger';
    case SyncDialogType.Offline':
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
  () => props.status,
  newStatus => {
    if (newStatus) {
      console.log('Sync progress event:', newStatus);
      syncStore.setStatus(newStatus); // Update the sync store with the new status
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

### 3. Update `Spinner.vue` to Read from the Store

Modify `Spinner.vue` to read the sync state from the Pinia store and update its message reactively.

```vue
<template>
  <div class="item default">
    <div class="header">
      <span class="title semi-bold">{{ titleText }}</span>
      <app-icon-button
        v-if="dismissable"
        :title="t('common.dismiss')"
        :name="'times'"
        class="icon-button dismiss"
        @click="dismiss"
      />
    </div>
    <div class="progress-info">
      {{ progressMessage }}
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
import { useSyncStore } from '@/stores/syncStore'; // Import the sync store

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
    dismiss

();
  }
};

const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;

const syncStore = useSyncStore(); // Use the sync store

const progressMessage = computed(() => {
  if (syncStore.status) {
    return `${syncStore.status.syncedModels.length}/${syncStore.status.allModels.length} models are done.`;
  }
  return '';
});

onMounted(() => {
  console.log('Spinner mounted and listening for events');
  toastService.on(ToastEvent.Dismiss, onDismiss);
});

onBeforeUnmount(() => {
  console.log('Spinner before unmount and removing event listeners');
  toastService.off(ToastEvent.Dismiss, onDismiss);
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

With these changes:

1. `Sync.vue` will update the Pinia store with the sync progress status.
2. `Spinner.vue` will read from the Pinia store and display the progress message reactively.

This approach ensures a reactive and centralized state management, making it easier to manage and debug the synchronization progress across components.
