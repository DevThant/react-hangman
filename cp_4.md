To resolve this issue, let's debug the steps to ensure that the progress events are being emitted correctly from `Sync.vue` and received properly by `Spinner.vue`. Here’s how to troubleshoot and ensure everything is wired correctly:

1. **Verify `Sync.vue` is Emitting Events:**

   Ensure that `Sync.vue` is emitting the progress events correctly.

   ```vue
   <script setup lang="ts">
   import { watch, computed } from "vue";
   import { PickupPaths, useI18n } from "vue-i18n";
   import { exhaustiveTypeCheck } from "@ebitoolmx/predicates";
   import AppIcon from "@/components/common/icon/Icon.vue";
   import AppButton from "@/components/common/formElements/button/Button.vue";
   import AppDialogContainer from "@/components/common/dialog/layouts/DialogContainer.vue";
   import AppDialogIconContentLayout, { IconColor } from "@/components/common/dialog/layouts/DialogIconContentLayout.vue";
   import AppDialogProcessContentLayout from "@/components/common/dialog/layouts/DialogProcessContentLayout.vue";
   import { SyncDialogType } from "@/typings/sync.js";
   import { DialogNames } from "@/typings/dialog.js";
   import { useAppStore } from "@/stores/app.js";
   import { useProductsStore } from "@/stores/products.js";
   import { useConnectionsStore } from "@/stores/connections.js";
   import { useEditorStore } from "@/stores/editor.js";
   import { eventService, EventType } from "@/services/event.js";
   import { LocaleMessage } from "@/locale/en.js";
   import { MXSyncProgressEvent, MXSyncProgressStatus } from "@ebitoolmx/gateway-types";
   import { useAuthService } from "@/auth/index.js";
   import eventBus from '@/services/eventBus'; // Import the event bus

   defineOptions({ name: "Sync" });
   const props = defineProps<{ dialogType: SyncDialogType; status?: MXSyncProgressEvent; }>();
   const emit = defineEmits(["close"]);
   const appStore = useAppStore();
   const { userDetails } = useAuthService();
   const connectionsStore = useConnectionsStore();
   const productsStore = useProductsStore();
   const editorStore = useEditorStore();
   const { t } = useI18n();

   const header = computed<PickupPaths<LocaleMessage>>(() => {
     switch (props.dialogType) {
       case SyncDialogType.Requested:
         return "sync.modal.header.requested";
       case SyncDialogType.Initialising:
         return "sync.modal.header.initialising";
       case SyncDialogType.Processing:
         return "sync.modal.header.processing";
       case SyncDialogType.Successful:
         return "sync.modal.header.successful";
       case SyncDialogType.InitialisingFailed:
         return "sync.modal.header.initialisingFailed";
       case SyncDialogType.Unsuccessful:
         return "sync.modal.header.unsuccessful";
       case SyncDialogType.Offline:
         return "sync.modal.header.offline";
       default:
         return exhaustiveTypeCheck(props.dialogType);
     }
   });

   const message = computed<PickupPaths<LocaleMessage> | null>(() => {
     switch (props.dialogType) {
       case SyncDialogType.Requested:
         return "sync.modal.message.requested";
       case SyncDialogType.Processing:
         return "sync.modal.message.progress";
       case SyncDialogType.Offline:
         return "sync.modal.message.offline";
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
       return "sync_success";
     }
     if (props.status?.currentModelName === model) {
       return "sync_in_progress";
     }
     return "sync_no_progress";
   };

   const icon = computed<string>(() => {
     switch (props.dialogType) {
       case SyncDialogType.Requested:
         return "sync";
       case SyncDialogType.Successful:
         return "sync_success";
       case SyncDialogType.Unsuccessful:
         return "warning";
       case SyncDialogType.InitialisingFailed:
         return "warning";
       case SyncDialogType.Offline:
         return "cloud-offline";
       default:
         return "";
     }
   });

   const iconColor = computed<IconColor>(() => {
     switch (props.dialogType) {
       case SyncDialogType.Requested:
         return "text";
       case SyncDialogType.Successful:
         return "success";
       case SyncDialogType.Unsuccessful:
         return "danger";
       case SyncDialogType.InitialisingFailed:
         return "danger";
       case SyncDialogType.Offline:
         return "warning";
       default:
         return "text";
     }
   });

   const isSyncRequestDialog = computed<boolean>(
     () => props.dialogType === SyncDialogType.Requested
   );

   const isSyncProcessingDialog = computed<boolean>(
     () => props.dialogType === SyncDialogType.Processing || props.dialogType === SyncDialogType.Initialising
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
     () => appStore.hasBlockingTasks || props.status?.syncProgressStatus === MXSyncProgressStatus.Synchronizing
   );

   function start() {
     const email = userDetails.value?.email;
     if (!email) return;

     productsStore.syncProduct({
       productId: productsStore.activeProductId,
       userId: email,
       editorType: editorStore.editorType,
     });
   }

   watch(() => props.status, (newStatus) => {
     if (newStatus) {
       console.log('Sync progress event:', newStatus); // Debugging log
       eventBus.emit('syncProgress', newStatus); // Emit the sync progress event
     }
   }, { immediate: true });

   watch(
     () => connectionsStore.isOffline,
     (nowOffline) => {
       if (nowOffline)
         eventService.emit(EventType.OpenDialog, {
           dialogName: DialogNames.Sync,
           options: {
             props: {
               dialogType: SyncDialogType.Offline,
             },
             modal: true,
           },
         });
     }
   );
   </script>
   ```

2. **Verify `Spinner.vue` is Receiving Events:**

   Update `Spinner.vue` to log events and ensure it receives the emitted events:

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
   import { ref, onBeforeUnmount, onMounted } from 'vue';

   import AppIconButton from '@/components/common/icon/IconButton.vue';
   import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';

   import { ToastEvent, toastService } from '@/services/toast.js';
   import { useI18n } from 'vue-i18n';
   import eventBus from '@/services/eventBus'; // Import the event bus
   import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types'; // Correct import

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

   const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;

   const progressMessage = ref('');

   const updateProgressMessage = (status: MXSyncProgressEvent) => {
     console.log('Received sync progress event:', status); // Debugging log
     progressMessage.value = `${status.syncedModels.length}/${status.allModels.length} models are done.`;
   };

   onMounted(() => {
     toastService.on(ToastEvent.Dismiss, onDismiss);
     eventBus.on('syncProgress', updateProgressMessage); // Listen for sync progress events
   });

   onBeforeUnmount(() => {
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
     width: 16

px;
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

   .header .dismiss:hover {
     fill: var(--toaster-dismiss-icon);
   }

   .progress-info {
     margin-top: 10px;
   }

   .spinner {
     margin: 15px 0 15px;
   }
   </style>
   ```

### Debugging Steps:

1. **Check Console Logs in `Sync.vue`:**
   - Ensure the log statement inside `watch` is printing the sync progress event data.

2. **Check Console Logs in `Spinner.vue`:**
   - Ensure the log statement inside `updateProgressMessage` is printing the received sync progress event data.

3. **Ensure Event Bus is Functioning:**
   - Verify that events are being emitted and received correctly.

With these steps, you should be able to see if the events are being emitted and received as expected. If the logs in both components show the expected data, but the UI does not update, it might indicate an issue with the reactivity in the `progressMessage` ref. 

---

**Sponsored**  
“ChatGPT for [Stock Market](https://api.adzedek.com/click_stockmarketgpt0314?chatbot_id=1715191360448x620213882279166000&operation_hash=e614b43d2bff2e511b7574c65c8737c0) Investors”

------------------

Given that `Sync.vue` already has precise data about the syncing progress, we can leverage this information to update the `Spinner.vue` component in real-time without making changes to the backend. 

Here’s how we can achieve this:

1. **Leverage `Sync.vue` to Emit Progress Events:**
   - Emit progress events whenever the sync status is updated.

2. **Use an Event Bus to Relay Updates:**
   - Create an event bus using a library like `mitt` to relay progress updates from `Sync.vue` to `Spinner.vue`.

3. **Update `Spinner.vue` to Listen for Progress Events:**
   - Update `Spinner.vue` to listen for these events and update the displayed progress accordingly.

### Step-by-Step Implementation

#### 1. Create an Event Bus

First, create an event bus that will allow `Sync.vue` to emit progress updates and `Spinner.vue` to listen to them.

Create a new file `eventBus.ts`:

```typescript
import mitt from 'mitt';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

type Events = {
  syncProgress: MXSyncProgressEvent;
};

const eventBus = mitt<Events>();

export default eventBus;
```

#### 2. Update `Sync.vue` to Emit Progress Events

Modify `Sync.vue` to emit progress events whenever the sync status is updated:

```vue
<script setup lang="ts">
import { watch, computed } from "vue";
import { PickupPaths, useI18n } from "vue-i18n";
import { exhaustiveTypeCheck } from "@ebitoolmx/predicates";
import AppIcon from "@/components/common/icon/Icon.vue";
import AppButton from "@/components/common/formElements/button/Button.vue";
import AppDialogContainer from "@/components/common/dialog/layouts/DialogContainer.vue";
import AppDialogIconContentLayout, { IconColor } from "@/components/common/dialog/layouts/DialogIconContentLayout.vue";
import AppDialogProcessContentLayout from "@/components/common/dialog/layouts/DialogProcessContentLayout.vue";
import { SyncDialogType } from "@/typings/sync.js";
import { DialogNames } from "@/typings/dialog.js";
import { useAppStore } from "@/stores/app.js";
import { useProductsStore } from "@/stores/products.js";
import { useConnectionsStore } from "@/stores/connections.js";
import { useEditorStore } from "@/stores/editor.js";
import { eventService, EventType } from "@/services/event.js";
import { LocaleMessage } from "@/locale/en.js";
import { MXSyncProgressEvent, MXSyncProgressStatus } from "@ebitoolmx/gateway-types";
import { useAuthService } from "@/auth/index.js";
import eventBus from '@/services/eventBus'; // Import the event bus

defineOptions({ name: "Sync" });
const props = defineProps<{ dialogType: SyncDialogType; status?: MXSyncProgressEvent; }>();
const emit = defineEmits(["close"]);
const appStore = useAppStore();
const { userDetails } = useAuthService();
const connectionsStore = useConnectionsStore();
const productsStore = useProductsStore();
const editorStore = useEditorStore();
const { t } = useI18n();

const header = computed<PickupPaths<LocaleMessage>>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return "sync.modal.header.requested";
    case SyncDialogType.Initialising:
      return "sync.modal.header.initialising";
    case SyncDialogType.Processing:
      return "sync.modal.header.processing";
    case SyncDialogType.Successful:
      return "sync.modal.header.successful";
    case SyncDialogType.InitialisingFailed:
      return "sync.modal.header.initialisingFailed";
    case SyncDialogType.Unsuccessful:
      return "sync.modal.header.unsuccessful";
    case SyncDialogType.Offline:
      return "sync.modal.header.offline";
    default:
      return exhaustiveTypeCheck(props.dialogType);
  }
});

const message = computed<PickupPaths<LocaleMessage> | null>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return "sync.modal.message.requested";
    case SyncDialogType.Processing:
      return "sync.modal.message.progress";
    case SyncDialogType.Offline:
      return "sync.modal.message.offline";
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
    return "sync_success";
  }
  if (props.status?.currentModelName === model) {
    return "sync_in_progress";
  }
  return "sync_no_progress";
};

const icon = computed<string>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return "sync";
    case SyncDialogType.Successful:
      return "sync_success";
    case SyncDialogType.Unsuccessful:
      return "warning";
    case SyncDialogType.InitialisingFailed:
      return "warning";
    case SyncDialogType.Offline:
      return "cloud-offline";
    default:
      return "";
  }
});

const iconColor = computed<IconColor>(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return "text";
    case SyncDialogType.Successful:
      return "success";
    case SyncDialogType.Unsuccessful:
      return "danger";
    case SyncDialogType.InitialisingFailed:
      return "danger";
    case SyncDialogType.Offline:
      return "warning";
    default:
      return "text";
  }
});

const isSyncRequestDialog = computed<boolean>(
  () => props.dialogType === SyncDialogType.Requested
);

const isSyncProcessingDialog = computed<boolean>(
  () => props.dialogType === SyncDialogType.Processing || props.dialogType === SyncDialogType.Initialising
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
  () => appStore.hasBlockingTasks || props.status?.syncProgressStatus === MXSyncProgressStatus.Synchronizing
);

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType,
  });
}

watch(() => props.status, (newStatus) => {
  if (newStatus) {
    eventBus.emit('syncProgress', newStatus); // Emit the sync progress event
  }
}, { immediate: true });

// Remaining code unchanged
</script>
```

#### 3. Update `Spinner.vue` to Listen for Progress Events

Modify `Spinner.vue` to listen for these progress events and update the displayed progress:

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
    <app-horizontal-spinner
      class="spinner"
      :thin="true"
    ></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from "vue";
import AppIconButton from "@/components/common/icon/IconButton.vue";
import AppHorizontalSpinner from "@/components/common/spinner/HorizontalSpinner.vue";
import { toastService, ToastEvent } from "@/services/toast.js";
import { useI18n } from "vue-i18n";
import eventBus from '@/services/eventBus'; // Import the event bus

defineOptions({ name: "SpinnerToast" });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
  }
);

const emit = defineEmits<{ dismiss: [id: symbol] }>();

const { t, te } = useI18n();

const dismiss = () => emit("dismiss", props.id);

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const titleText: string = te(props.title)
  ? t(props.title, props.titleVars)
  : props.title;

const progressMessage = ref('');

const updateProgressMessage = (status: MXSyncProgressEvent) => {
  progressMessage.value = `${status.syncedModels.length}/${status.allModels.length} models are done.`;
};

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  eventBus.on('syncProgress', updateProgressMessage); // Listen for sync progress events
});

onBeforeUnmount(() => {
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

.header .dismiss:hover {
  fill: var(--toaster-dismiss-icon);
}

.progress-info {
  margin-top: 10px;
}

.spinner {
  margin: 15px 0 15px;
}
</style>
```

With these changes, the `Spinner.vue` component will now display the real-time progress of the synchronization process, updating the number of models synced in real-time without requiring any changes to the backend.

---

**Sponsored**  
“ChatGPT for [Stock Market](https://api.adzedek.com/click_stockmarketgpt0314?chatbot_id=1715191360448x620213882279166000&operation_hash=e614b43d2bff2e511b7574c65c8737c0) Investors”
