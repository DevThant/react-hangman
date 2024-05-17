Let's debug and refine the implementation step by step to ensure everything works correctly. Here's a detailed plan:

1. **Ensure `Sync.vue` emits the correct progress and totalModels values.**
2. **Verify the toast service updates the `SpinnerToast` component correctly.**
3. **Ensure the `SpinnerToast` component displays the progress and totalModels correctly.**

We'll start by adding console logs to each step to track where the issue might be.

### Step 1: Ensure `Sync.vue` emits the correct progress and totalModels values

Update the `start` function in `Sync.vue` to log the progress and totalModels values:

```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { useI18n, PickupPaths } from 'vue-i18n';

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

import { toastService } from '@/services/toast.js';

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

  // Emit progress update to the toastService
  const toastId = Symbol('sync-toast');
  const totalModels = props.status?.allModels.length || 0;
  toastService.displaySpinner(toastId, t('sync.spinnerTitle'), 0, totalModels);

  const interval = setInterval(() => {
    const progress = props.status?.syncedModels.length || 0;
    console.log(`Progress: ${progress}, Total Models: ${totalModels}`); // Add log for progress and totalModels
    toastService.displaySpinner(toastId, t('sync.spinnerTitle'), progress, totalModels);
    
    if (progress >= totalModels) {
      clearInterval(interval);
      toastService.dismiss(toastId);
    }
  }, 1000);
}

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

### Step 2: Verify the toast service updates the `SpinnerToast` component correctly

Add logs in `toast.ts` to ensure it's receiving the correct values:

```ts
public displaySpinner(id: symbol, header: string, progress: number, totalModels: number): symbol {
  console.log(`Displaying spinner: ${header}, Progress: ${progress}, Total Models: ${totalModels}`); // Add log for spinner display
  return this.displayWithId(id, ToastType.Spinner, { title: header, progress, totalModels });
}
```

### Step 3: Ensure the `SpinnerToast` component displays the progress and totalModels correctly

Add logs in `SpinnerToast.vue` to verify it receives the correct props:

```vue
<script setup lang="ts">
import { onBeforeUnmount, onMounted } from 'vue';
import { useI18n } from 'vue-i18n';

import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';

import { ToastEvent, toastService } from '@/services/toast.js';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    progress: number;
    totalModels: number;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    progress: 0,
    totalModels: 0
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

const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  console.log(`Mounted SpinnerToast with Progress: ${props.progress}, Total Models: ${props.totalModels}`); // Add log for mounted props
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});
</script>
```

### Adding i18n Keys

Ensure the i18n keys are correctly defined in your translation files.

**en.json**:
```json
{
  "common": {
    "dismiss": "Dismiss",
    "start": "Start",
    "close": "Close",
    "help
    
``` 

--------


Based on the logs showing `0 progress` and `0 totalModels` throughout the sync, it seems the issue might be with the `props.status` not being updated correctly. Let's go through the steps to ensure `props.status` is properly set and updated during the synchronization process.

### Step-by-Step Debugging

1. **Check the Initialization of `props.status` in `Sync.vue`:**
   - Ensure that `props.status` is properly initialized and contains the correct data structure when the synchronization starts.
   
2. **Verify Updates to `props.status`:**
   - Ensure that `props.status` is being updated correctly during the synchronization process.

### Update `Sync.vue` to Track Status Changes

Let's add more logs and checks to ensure `props.status` is updated correctly. We will also add watchers to track changes in `props.status`.

**Sync.vue:**

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
          <div v-else-if="isSyncSuccessfulDialog" class="successful">
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
import { useI18n, PickupPaths } from 'vue-i18n';

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

import { toastService } from '@/services/toast.js';

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

watch(
  () => props.status,
  (newStatus, oldStatus) => {
    console.log('Status changed:', newStatus); // Log status changes
  },
  { immediate: true, deep: true }
);

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Emit progress update to the toastService
  const toastId =Symbol(‘sync-toast’);
const totalModels = props.status?.allModels.length || 0;
toastService.displaySpinner(toastId, t(‘sync.spinnerTitle’), 0, totalModels);

const interval = setInterval(() => {
const progress = props.status?.syncedModels.length || 0;
console.log(Progress: ${progress}, Total Models: ${totalModels}); // Add log for progress and totalModels
toastService.displaySpinner(toastId,t(‘sync.spinnerTitle’), progress, totalModels);
```
 Symbol('sync-toast');
  const totalModels = props.status?.allModels.length || 0;
  toastService.displaySpinner(toastId, t('sync.spinnerTitle'), 0, totalModels);

  const interval = setInterval(() => {
    const progress = props.status?.syncedModels.length || 0;
    console.log(`Progress: ${progress}, Total Models: ${totalModels}`); // Add log for progress and totalModels
    toastService.displaySpinner(toastId, t('sync.spinnerTitle'), progress, totalModels);
    
    if (progress >= totalModels) {
      clearInterval(interval);
      toastService.dismiss(toastId);
    }
  }, 1000);
}

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
    color: var (--success);
    fill: var (--success);
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

.successful {
  display: flex;
  flex-direction: column;
  align-items: center;

  ul {
    list-style: none;
    padding: 0;
    margin: 0;

    li {
      display: flex;
      align-items: center;
      margin-bottom: var(--base-spacing-1);

      .icon {
        margin-right: var(--base-spacing-1);
      }
    }
  }
}
</style>
```

### Verify `props.status` Data Structure

Ensure that `props.status` has the following structure:

```ts
interface MXSyncProgressEvent {
  allModels: string[];
  syncedModels: string[];
  currentModelName: string;
  syncProgressStatus: MXSyncProgressStatus;
}
```

Verify that `allModels` and `syncedModels` are arrays and that they are updated correctly during the synchronization process.

### Update the Sync Process to Properly Update `props.status`

Ensure the sync process properly updates `props.status`. If you're using a store to manage synchronization state, ensure that the store correctly updates `props.status`.

If the problem persists, please share how `props.status` is being updated in the sync process, and we'll further refine the solution.

