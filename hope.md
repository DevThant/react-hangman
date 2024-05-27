Yes, you can pass the `allModels` and `syncedModels` data to the `Spinner.vue` component from `useSync.ts`. Since `useSync.ts` already handles the WebSocket subscription and synchronization status, you can modify the `toastService.display` calls to include this data as additional properties.

### Steps to Pass Data from `useSync.ts` to `Spinner.vue`

1. **Update `toastService.display` in `useSync.ts`:**
   - Add `allModels` and `syncedModels` to the `toastService.display` call when creating the spinner toast.

2. **Modify `Spinner.vue` to Accept New Props:**
   - Update `Spinner.vue` to accept `allModels` and `syncedModels` as props.

3. **Update Toast Service Handling:**
   - Ensure the toast service can handle and pass these new props to the `Spinner.vue` component.

### High-Level Steps:

1. **Update `useSync.ts` for Spinner Toast:**
   - When creating the spinner toast, include `allModels` and `syncedModels` in the properties passed to `toastService.display`.

   ```javascript
   if ((syncProcessing || syncInitialising) && !syncProcessingId.value) {
     syncProcessingId.value = toastService.display(ToastType.Spinner, {
       type: 'default',
       title: 'sync.toaster.requested',
       titleVars: {
         syncedBy: newStatus.synchronizationProgress?.syncedBy
       },
       allModels: newStatus.synchronizationProgress?.allModels,
       syncedModels: newStatus.synchronizationProgress?.syncedModels,
       dismissable: true
     });
   }
   ```

2. **Modify `Spinner.vue` to Accept Props:**
   - Update `Spinner.vue` to accept `allModels` and `syncedModels` as props.

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
       <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
       <div v-if="allModels && syncedModels">
         <p>All Models: {{ allModels.join(', ') }}</p>
         <p>Synced Models: {{ syncedModels.join(', ') }}</p>
       </div>
     </div>
   </template>

   <script setup lang="ts">
   import { onBeforeUnmount, onMounted } from 'vue';

   import AppIconButton from '@/components/common/icon/IconButton.vue';
   import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';

   import { ToastEvent, toastService } from '@/services/toast.js';
   import { useI18n } from 'vue-i18n';

   defineOptions({ name: 'SpinnerToast' });

   const props = withDefaults(
     defineProps<{
       id: symbol;
       title: string;
       titleVars?: Record<string, unknown>;
       dismissable?: boolean;
       allModels?: string[];
       syncedModels?: string[];
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

   onMounted(() => {
     toastService.on(ToastEvent.Dismiss, onDismiss);
   });

   onBeforeUnmount(() => {
     toastService.off(ToastEvent.Dismiss, onDismiss);
   });
   </script>

   <style scoped>
   /* styles */
   </style>
   ```

3. **Ensure Toast Service Can Handle New Props:**
   - Verify that the toast service can pass the `allModels` and `syncedModels` props to `Spinner.vue`.

By following these steps, you can ensure that `Spinner.vue` receives the necessary synchronization data (`allModels` and `syncedModels`) directly from `useSync.ts` via the toast service. This approach ensures real-time synchronization data is displayed in the spinner toast for all users, even if they are on different networks and devices.


-----------

To ensure that the Toast Service can handle the new props (`allModels` and `syncedModels`) and pass them to the `Spinner.vue` component, you'll need to update the `ToastService` to pass these properties correctly and modify the `Toaster.vue` component to accept and pass these properties to the spinner component.

### Step-by-Step Solution:

1. **Update `ToastService` to Include New Props:**
   - Modify the `displayWithId` method to handle additional props for `allModels` and `syncedModels`.

   ```typescript
   public displayWithId(
     id: symbol,
     type: ToastType,
     props: Record<string, unknown> = {},
     timeout: number | undefined = undefined
   ): symbol {
     const properties = { ...props };
     const component = this.toastComponents.get(type);

     if (!component) this.loggingService.error(`Toast type '${type}' has not been registered.`);

     this.events.emit(ToastEvent.Display, { id, component, properties, timeout });
     this.loggingService.debug(`Displaying toast of type '${type}'`);

     return id;
   }

   public display(
     type: ToastType,
     props: Record<string, unknown> = {},
     timeout: number | undefined = undefined
   ): symbol {
     return this.displayWithId(Symbol('toast'), type, props, timeout);
   }
   ```

2. **Update `Spinner.vue` to Accept New Props:**
   - Ensure that `Spinner.vue` can receive and display the new properties.

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
       <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
       <div v-if="allModels && syncedModels">
         <p>All Models: {{ allModels.join(', ') }}</p>
         <p>Synced Models: {{ syncedModels.join(', ') }}</p>
       </div>
     </div>
   </template>

   <script setup lang="ts">
   import { onBeforeUnmount, onMounted } from 'vue';

   import AppIconButton from '@/components/common/icon/IconButton.vue';
   import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';

   import { ToastEvent, toastService } from '@/services/toast.js';
   import { useI18n } from 'vue-i18n';

   defineOptions({ name: 'SpinnerToast' });

   const props = withDefaults(
     defineProps<{
       id: symbol;
       title: string;
       titleVars?: Record<string, unknown>;
       dismissable?: boolean;
       allModels?: string[];
       syncedModels?: string[];
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

   onMounted(() => {
     toastService.on(ToastEvent.Dismiss, onDismiss);
   });

   onBeforeUnmount(() => {
     toastService.off(ToastEvent.Dismiss, onDismiss);
   });
   </script>

   <style scoped>
   /* styles */
   </style>
   ```

3. **Update `Toaster.vue` to Handle New Props:**
   - Ensure that `Toaster.vue` can handle the new properties and pass them to the components.

   ```vue
   <template>
     <transition-group
       v-if="displayedToasts.length"
       name="list"
       tag="div"
       class="container lead"
       data-testid="toaster"
     >
       <component
         :is="toast.component"
         v-for="toast in displayedToasts"
         :key="toast.id"
         v-bind="{ id: toast.id, class: 'toast', ...toast.properties }"
         @click="toast.properties.onClick"
         @dismiss="removeToast"
       />
     </transition-group>
   </template>

   <script setup lang="ts">
   import { onBeforeUnmount, onMounted, ref, shallowRef } from 'vue';

   import { toastService, ToastEvent, Toast } from '@/services/toast.js';

   defineOptions({ name: 'Toaster' });

   const displayedToasts = shallowRef<Toast[]>([]);
   const toastTimeouts = ref<Map<symbol, number>>(new Map());

   const removeToast = (id: symbol): void => {
     if (toastTimeouts.value.has(id)) {
       window.clearTimeout(toastTimeouts.value.get(id));
       toastTimeouts.value.delete(id);
     }

     displayedToasts.value = displayedToasts.value.filter(toast => toast.id !== id);
   };

   const displayToast = (toast: Toast): void => {
     displayedToasts.value = [...displayedToasts.value, toast];

     if (toast.timeout) {
       const timeoutHandle = window.setTimeout(() => {
         removeToast(toast.id);
       }, toast.timeout);

       toastTimeouts.value.set(toast.id, timeoutHandle);
     }
   };

   const clearAllToasts = () => {
     displayedToasts.value = [];
   };

   onMounted(() => {
     toastService.on(ToastEvent.Display, displayToast);
     toastService.on(ToastEvent.CloseAll, clearAllToasts);
   });

   onBeforeUnmount(() => {
     toastService.off(ToastEvent.Display, displayToast);
     toastService.off(ToastEvent.CloseAll, clearAllToasts);
   });
   </script>

   <style scoped>
   /* styles */
   </style>
   ```

This ensures that the `ToastService` can handle and pass the `allModels` and `syncedModels` props to the `Spinner.vue` component through `Toaster.vue`.

Do you have any specific test cases or scenarios in mind to ensure that the spinner toast displays the correct synchronization data?
---------------------


Based on the logs, it appears that `allModels` and `syncedModels` data are being passed inconsistently. Since `sync.vue` achieves live updates, let's ensure that the spinner toast can also update reactively.

Here are the steps to ensure reactive updates for `Spinner.vue`:

### 1. Ensure Reactive Data Handling in `useSync.ts`

Make sure `syncStatusSubscription` is correctly set up as a reactive reference and updates its state:

```typescript
watch(
  [syncStatusSubscription, () => isSyncEnabled.value],
  ([newStatus], [oldStatus]) => {
    if (!newStatus || !isSyncEnabled.value) {
      appStore.removeBlockingTask(taskId.value);
      return;
    }

    const newSyncStatus = newStatus.synchronizationProgress?.syncProgressStatus;
    const oldSyncStatus = oldStatus?.synchronizationProgress?.syncProgressStatus;

    const syncedByUser = newStatus.synchronizationProgress?.syncedBy === userDetails.value?.email;
    const syncProcessing = newSyncStatus === MXSyncProgressStatus.Synchronizing;

    const syncInitialising = newSyncStatus === MXSyncProgressStatus.Initializing;

    const syncSuccessful =
      (newSyncStatus === MXSyncProgressStatus.Successful &&
        oldSyncStatus === MXSyncProgressStatus.Synchronizing) ||
      (newSyncStatus === MXSyncProgressStatus.Successful &&
        oldSyncStatus === MXSyncProgressStatus.Initializing);

    const syncUnsuccessful =
      newSyncStatus === MXSyncProgressStatus.Unsuccessful &&
      oldSyncStatus === MXSyncProgressStatus.Synchronizing;

    const syncInitialisationFailed =
      newSyncStatus === MXSyncProgressStatus.InitializationFailed &&
      oldSyncStatus === MXSyncProgressStatus.Initializing;

    if (syncProcessing) {
      appStore.addBlockingTask(taskId.value);
    } else {
      appStore.removeBlockingTask(taskId.value);
      if (syncProcessingId.value) {
        toastService.dismiss(syncProcessingId.value);
        syncProcessingId.value = null;
      }
    }

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
      console.log('new status', newStatus.synchronizationProgress);
      syncProcessingId.value = toastService.display(ToastType.Spinner, {
        type: 'default',
        title: 'sync.toaster.requested',
        titleVars: {
          syncedBy: newStatus.synchronizationProgress?.syncedBy
        },
        allModels: newStatus.synchronizationProgress?.allModels || [],
        syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
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
```

### 2. Ensure `Toaster.vue` Can Handle Dynamic Updates

Make sure `Toaster.vue` is correctly set up to pass dynamic props to the toast components. It looks fine, but ensure no issues in prop passing.

### 3. Make `Spinner.vue` Reactive to Props

Ensure `Spinner.vue` is reactive and updates whenever `allModels` or `syncedModels` props change.

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
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
    <div v-if="allModels && allModels.length && syncedModels && syncedModels.length">
      <p>All Models: {{ allModels.join(', ') }}</p>
      <p>Synced Models: {{ syncedModels.join(', ') }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, watch } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    allModels?: string[];
    syncedModels?: string[];
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    allModels: [],
    syncedModels: []
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
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});

watch(
  () => props.allModels,
  (newModels) => {
    console.log('Updated allModels:', newModels);
  },
  { immediate: true }
);

watch(
  () => props.syncedModels,
  (newModels) => {
    console.log('Updated syncedModels:', newModels);
  },
  { immediate: true }
);
</script>

<style scoped>
/* styles */
</style>
```

### 4. Synchronize State Across Components

Ensure the state is synchronized and consistent across all components:

1. **Ensure `useSync.ts` Always Sends Data:**
   - Make sure the data is always being sent and logged correctly.

2. **React to Prop Changes in `Spinner.vue`:**
   - Use watchers in `Spinner.vue` to reactively update and log changes to props.

### Summary

- **Reactive Data Handling:**
  - Ensure `useSync.ts` properly watches and passes reactive data.
- **Prop Handling:**
  - Ensure `Spinner.vue` is set up to reactively update based on prop changes.
- **Consistency:**
  - Use logging to ensure consistency and trace data flow.

By following these steps, you can ensure that `Spinner.vue` correctly receives and updates with the `allModels` and `syncedModels` data consistently. If you continue to see issues, check for any potential race conditions or timing problems that might affect the data flow.

-----


Based on the logs, it appears that `allModels` and `syncedModels` data are being passed inconsistently. Since `sync.vue` achieves live updates, let's ensure that the spinner toast can also update reactively.

Here are the steps to ensure reactive updates for `Spinner.vue`:

### 1. Ensure Reactive Data Handling in `useSync.ts`

Make sure `syncStatusSubscription` is correctly set up as a reactive reference and updates its state:

```typescript
watch(
  [syncStatusSubscription, () => isSyncEnabled.value],
  ([newStatus], [oldStatus]) => {
    if (!newStatus || !isSyncEnabled.value) {
      appStore.removeBlockingTask(taskId.value);
      return;
    }

    const newSyncStatus = newStatus.synchronizationProgress?.syncProgressStatus;
    const oldSyncStatus = oldStatus?.synchronizationProgress?.syncProgressStatus;

    const syncedByUser = newStatus.synchronizationProgress?.syncedBy === userDetails.value?.email;
    const syncProcessing = newSyncStatus === MXSyncProgressStatus.Synchronizing;

    const syncInitialising = newSyncStatus === MXSyncProgressStatus.Initializing;

    const syncSuccessful =
      (newSyncStatus === MXSyncProgressStatus.Successful &&
        oldSyncStatus === MXSyncProgressStatus.Synchronizing) ||
      (newSyncStatus === MXSyncProgressStatus.Successful &&
        oldSyncStatus === MXSyncProgressStatus.Initializing);

    const syncUnsuccessful =
      newSyncStatus === MXSyncProgressStatus.Unsuccessful &&
      oldSyncStatus === MXSyncProgressStatus.Synchronizing;

    const syncInitialisationFailed =
      newSyncStatus === MXSyncProgressStatus.InitializationFailed &&
      oldSyncStatus === MXSyncProgressStatus.Initializing;

    if (syncProcessing) {
      appStore.addBlockingTask(taskId.value);
    } else {
      appStore.removeBlockingTask(taskId.value);
      if (syncProcessingId.value) {
        toastService.dismiss(syncProcessingId.value);
        syncProcessingId.value = null;
      }
    }

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
      console.log('new status', newStatus.synchronizationProgress);
      syncProcessingId.value = toastService.display(ToastType.Spinner, {
        type: 'default',
        title: 'sync.toaster.requested',
        titleVars: {
          syncedBy: newStatus.synchronizationProgress?.syncedBy
        },
        allModels: newStatus.synchronizationProgress?.allModels || [],
        syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
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
```

### 2. Ensure `Toaster.vue` Can Handle Dynamic Updates

Make sure `Toaster.vue` is correctly set up to pass dynamic props to the toast components. It looks fine, but ensure no issues in prop passing.

### 3. Make `Spinner.vue` Reactive to Props

Ensure `Spinner.vue` is reactive and updates whenever `allModels` or `syncedModels` props change.

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
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
    <div v-if="allModels && allModels.length && syncedModels && syncedModels.length">
      <p>All Models: {{ allModels.join(', ') }}</p>
      <p>Synced Models: {{ syncedModels.join(', ') }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, watch } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    allModels?: string[];
    syncedModels?: string[];
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    allModels: [],
    syncedModels: []
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
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});

watch(
  () => props.allModels,
  (newModels) => {
    console.log('Updated allModels:', newModels);
  },
  { immediate: true }
);

watch(
  () => props.syncedModels,
  (newModels) => {
    console.log('Updated syncedModels:', newModels);
  },
  { immediate: true }
);
</script>

<style scoped>
/* styles */
</style>
```

### 4. Synchronize State Across Components

Ensure the state is synchronized and consistent across all components:

1. **Ensure `useSync.ts` Always Sends Data:**
   - Make sure the data is always being sent and logged correctly.

2. **React to Prop Changes in `Spinner.vue`:**
   - Use watchers in `Spinner.vue` to reactively update and log changes to props.

### Summary

- **Reactive Data Handling:**
  - Ensure `useSync.ts` properly watches and passes reactive data.
- **Prop Handling:**
  - Ensure `Spinner.vue` is set up to reactively update based on prop changes.
- **Consistency:**
  - Use logging to ensure consistency and trace data flow.

By following these steps, you can ensure that `Spinner.vue` correctly receives and updates with the `allModels` and `syncedModels` data consistently. If you continue to see issues, check for any potential race conditions or timing problems that might affect the data flow.