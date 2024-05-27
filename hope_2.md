The issue you're encountering is due to a mismatch in the types expected by the `update` method in the `ToastService`. Specifically, the `update` method expects the first parameter to be a `symbol`, which is the ID of the toast, but you're passing an enum value (`ToastType.Spinner`).

To fix this, you need to keep track of the ID of the spinner toast when it's initially created, and then use that ID when calling the `update` method. Here's how you can do it:

### Full Code for `useSync.ts`

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
        if (syncProcessingId.value) {
          toastService.update(syncProcessingId.value, {
            allModels: newStatus.synchronizationProgress?.allModels || [],
            syncedModels: newStatus.synchronizationProgress?.syncedModels || []
          });
        }
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

  onBeforeUnmount(() => {
    if (syncProcessingId.value) toastService.dismiss(syncProcessingId.value);
    appStore.removeBlockingTask(taskId.value);
  });
}
```

### Explanation

1. **Track Toast ID:** We keep track of the spinner toast ID using `syncProcessingId.value`.
2. **Update Toast Props:** When the synchronization status is `SYNCHRONIZING`, we call the `update` method with the tracked toast ID and new properties.
3. **Display Toast:** When the synchronization is initializing or processing, and there is no existing spinner toast, we create a new one and save its ID.

By ensuring that the `update` method receives the correct `symbol` (the toast ID) and not an enum value, the code will correctly update the spinner toast with the new data, similar to how `Sync.vue` works.



---------------

It appears that TypeScript is not recognizing the `ToastEvent.Update` event because we didn't properly extend the type definitions in `ToastService`. Let’s correct that and make sure everything is consistent.

Here’s the revised implementation for the `ToastService` and the related parts:

### Step-by-Step Integration

#### 1. Update `ToastService`

Ensure the `ToastEvent` and `ToastService` are properly defined to handle updates:

```typescript
import { DefineComponent } from 'vue';
import mitt, { Emitter, Handler } from 'mitt';
import { LoggingService } from '@ebitoolmx/logging-service/console';
import { loggingService as commonLoggingService } from './logging.js';

export enum ToastEvent {
  Display = 'display',
  Dismiss = 'dismiss',
  CloseAll = 'closeAll',
  Update = 'update' // Add the Update event
}

export enum ToastType {
  Simple = 'simple',
  UpdateAvailable = 'updateAvailable',
  Spinner = 'spinner',
  Reconnecting = 'reconnecting',
  PickUpWhereYouLeftOff = 'pickUpWhereYouLeftOff'
}

export interface Toast {
  id: symbol;
  component?: DefineComponent;
  properties: Record<string, unknown>;
  timeout?: number;
}

type Events = {
  [ToastEvent.Display]: Toast;
  [ToastEvent.Dismiss]: symbol;
  [ToastEvent.CloseAll]: undefined;
  [ToastEvent.Update]: { id: symbol; props: Record<string, unknown> }; // Add Update event type
};

/**
 * Handles toaster events and components for the toaster component.
 */
export class ToastService {
  protected toastComponents = new Map<ToastType, DefineComponent>();

  protected events: Emitter<Events> = mitt<Events>();

  public constructor(protected loggingService: LoggingService) {}

  /**
   * Registers a toast into the service so that it can be referenced later when displaying
   */
  public registerToast(type: ToastType, toast: any) {
    this.toastComponents.set(type, toast);
  }

  /**
   * Displays a toast with a given id and of specified type. This will emit an event to the Toaster
   * component with the necessary details to display a toast. This is the most low-level endpoint
   * that all details to be able to display a toast.
   */
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
    this.loggingService.debug(`Displaying toast of type '${type}'`, properties);

    return id;
  }

  /**
   * Displays a toast of a specified type. This will auto generate an id and return it before
   * emitting an event to the Toaster component. See displayWithId().
   */
  public display(
    type: ToastType,
    props: Record<string, unknown> = {},
    timeout: number | undefined = undefined
  ): symbol {
    return this.displayWithId(Symbol('toast'), type, props, timeout);
  }

  /**
   * Updates the properties of an existing toast.
   */
  public update(id: symbol, props: Record<string, unknown>): void {
    this.events.emit(ToastEvent.Update, { id, props });
    this.loggingService.debug(`Updating toast with id '${id}'`, props);
  }

  /**
   * Dismisses a toast with the provided id if it is still present on the screen.
   */
  public dismiss(id: symbol): void {
    this.events.emit(ToastEvent.Dismiss, id);
  }

  /**
   * Closes all toasts
   */
  public closeAll(): void {
    this.events.emit(ToastEvent.CloseAll);
  }

  public on<Type extends ToastEvent.CloseAll>(type: Type, callback: Function): void;
  public on<Type extends ToastEvent.Dismiss>(type: Type, callback: Function): void;
  public on<Type extends ToastEvent.Display>(type: Type, callback: (toast: Toast) => void): void;
  public on<Type extends ToastEvent.Update>(type: Type, callback: (update: { id: symbol; props: Record<string, unknown> }) => void): void;
  public on<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.on(type, callback);
  }

  public off<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.off(type, callback);
  }
}

export const toastService = new ToastService(commonLoggingService);
```

#### 2. Update `Toaster.vue`

Ensure `Toaster.vue` properly listens for the `ToastEvent.Update` event and updates the displayed toasts accordingly:

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

const updateToast = ({ id, props }: { id: symbol; props: Record<string, unknown> }): void => {
  const toastIndex = displayedToasts.value.findIndex(toast => toast.id === id);
  if (toastIndex !== -1) {
    displayedToasts.value[toastIndex].properties = {
      ...displayedToasts.value[toastIndex].properties,
      ...props
    };
    // Trigger Vue reactivity by creating a new array
    displayedToasts.value = [...displayedToasts.value];
  }
};

const clearAllToasts = () => {
  displayedToasts.value = [];
};

onMounted(() => {
  toastService.on(ToastEvent.Display, displayToast);
  toastService.on(ToastEvent.CloseAll, clearAllToasts);
  toastService.on(ToastEvent.Update, updateToast);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Display, displayToast);
  toastService.off(ToastEvent.CloseAll, clearAllToasts);
  toastService.off(ToastEvent.Update, updateToast);
});
</script>

<style scoped>
/* styles */
</style>
```

### Summary

1. **Ensure `ToastService` Handles `ToastEvent.Update`:**
   - Define the `ToastEvent.Update` event and update method properly.

2. **Update `Toaster.vue`:**
   - Ensure `Toaster.vue` listens for the `ToastEvent.Update` event and updates the props of existing toasts accordingly.

By following these steps, you should be able to ensure that the spinner toast in `Spinner.vue` actively reflects the changes from `useSync.ts`, just like `Sync.vue`.


--------------

Let's fully integrate the `update` method into the `ToastService`. We'll need to:

1. Add the `update` method to the `ToastService`.
2. Define a new `ToastEvent.Update` event.
3. Ensure `Toaster.vue` listens for the `ToastEvent.Update` event and updates the displayed toasts accordingly.

### Step-by-Step Integration

#### 1. Update the `ToastService`

We'll add the `update` method and define the `ToastEvent.Update` event:

```typescript
import { DefineComponent } from 'vue';
import mitt, { Emitter, Handler } from 'mitt';
import { LoggingService } from '@ebitoolmx/logging-service/console';
import { loggingService as commonLoggingService } from './logging.js';

export enum ToastEvent {
  Display = 'display',
  Dismiss = 'dismiss',
  CloseAll = 'closeAll',
  Update = 'update' // Add the Update event
}

export enum ToastType {
  Simple = 'simple',
  UpdateAvailable = 'updateAvailable',
  Spinner = 'spinner',
  Reconnecting = 'reconnecting',
  PickUpWhereYouLeftOff = 'pickUpWhereYouLeftOff'
}

export interface Toast {
  id: symbol;
  component?: DefineComponent;
  properties: Record<string, unknown>;
  timeout?: number;
}

type Events = {
  [ToastEvent.Display]: Toast;
  [ToastEvent.Dismiss]: symbol;
  [ToastEvent.CloseAll]: undefined;
  [ToastEvent.Update]: { id: symbol; props: Record<string, unknown> }; // Add Update event type
};

export class ToastService {
  protected toastComponents = new Map<ToastType, DefineComponent>();

  protected events: Emitter<Events> = mitt<Events>();

  public constructor(protected loggingService: LoggingService) {}

  public registerToast(type: ToastType, toast: any) {
    this.toastComponents.set(type, toast);
  }

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
    this.loggingService.debug(`Displaying toast of type '${type}'`, properties);

    return id;
  }

  public display(
    type: ToastType,
    props: Record<string, unknown> = {},
    timeout: number | undefined = undefined
  ): symbol {
    return this.displayWithId(Symbol('toast'), type, props, timeout);
  }

  public update(id: symbol, props: Record<string, unknown>): void {
    this.events.emit(ToastEvent.Update, { id, props });
    this.loggingService.debug(`Updating toast with id '${id}'`, props);
  }

  public displayInfo(arg1: string, arg2?: string | number, arg3?: number): symbol {
    const header: string | undefined = typeof arg2 === 'string' ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === 'string' ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === 'number' ? arg2 : arg3;

    return this.display(ToastType.Simple, { title: header, text: message, type: 'info' }, timeout);
  }

  public displayWarning(arg1: string, arg2?: string | number, arg3?: number): symbol {
    const header: string | undefined = typeof arg2 === 'string' ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === 'string' ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === 'number' ? arg2 : arg3;

    return this.display(
      ToastType.Simple,
      { title: header, text: message, type: 'warning' },
      timeout
    );
  }

  public displayError(arg1: string, arg2?: string): symbol {
    const header = arg2 ? arg1 : undefined;
    const message = arg2 ?? arg1;

    return this.display(ToastType.Simple, { title: header, text: message, type: 'error' });
  }

  public displaySpinner(id: symbol, header: string): symbol {
    return this.displayWithId(id, ToastType.Spinner, { title: header });
  }

  public dismiss(id: symbol): void {
    this.events.emit(ToastEvent.Dismiss, id);
  }

  public closeAll(): void {
    this.events.emit(ToastEvent.CloseAll);
  }

  public on<Type extends ToastEvent.CloseAll>(type: Type, callback: Function): void;
  public on<Type extends ToastEvent.Dismiss>(type: Type, callback: Function): void;
  public on<Type extends ToastEvent.Display>(type: Type, callback: (toast: Toast) => void): void;
  public on<Type extends ToastEvent.Update>(type: Type, callback: (update: { id: symbol; props: Record<string, unknown> }) => void): void;
  public on<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.on(type, callback);
  }

  public off<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.off(type, callback);
  }
}

export const toastService = new ToastService(commonLoggingService);
```

#### 2. Update `Toaster.vue` to Handle Prop Changes

Add a method to `Toaster.vue` to handle the `ToastEvent.Update` event:

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
  console.log('Displaying toast:', toast);
  displayedToasts.value = [...displayedToasts.value, toast];

  if (toast.timeout) {
    const timeoutHandle = window.setTimeout(() => {
      removeToast(toast.id);
    }, toast.timeout);

    toastTimeouts.value.set(toast.id, timeoutHandle);
  }
};

const updateToast = ({ id, props }: { id: symbol; props: Record<string, unknown> }): void => {
  const toastIndex = displayedToasts.value.findIndex(toast => toast.id === id);
  if (toastIndex !== -1) {
    displayedToasts.value[toastIndex].properties = {
      ...displayedToasts.value[toastIndex].properties,
      ...props
    };
    // Trigger Vue reactivity by creating a new array
    displayedToasts.value = [...displayedToasts.value];
  }
};

const clearAllToasts = () => {
  displayedToasts.value = [];
};

onMounted(() => {
  toastService.on(ToastEvent.Display, displayToast);
  toastService.on(ToastEvent.CloseAll, clearAllToasts);
  toastService.on(ToastEvent.Update, updateToast);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Display, displayToast);
  toastService.off(ToastEvent.CloseAll, clearAllToasts);
  toastService.off(ToastEvent.Update, updateToast);
});
</script>

<style scoped>
/* styles */
</style>
```

### 3. Modify `Spinner.vue` to React to Prop Changes

Ensure that `Spinner.vue` can reactively handle prop updates:

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

### Summary

1. **Update `ToastService` to Handle Prop Changes:**
   - Add a method to update existing toasts and define a new `ToastEvent.Update` event.
   
2. **Ensure `Toaster.vue` Handles Updates:**
   - Add a method to handle the `ToastEvent.Update` event and update the props of existing toasts.

3. **Ensure `Spinner.vue` Reacts to Prop Changes:**
   - Ensure `Spinner.vue` properly reacts to updates in props.

By following these steps, you ensure that the `Spinner.vue` component within the toaster will actively reflect changes from `useSync.ts`, just like `Sync.vue`.


---------------

To ensure that the `Spinner.vue` component in the toaster actively reflects the changes from `useSync.ts`, similar to how `Sync.vue` does, we need to ensure the following:

1. **Reactive Data Flow:**
   - Ensure that `ToastService` updates the props for the spinner toast whenever `syncStatusSubscription` changes.
   
2. **Continuous Prop Updates:**
   - Make sure `Spinner.vue` can continuously receive updates and not just a one-time prop pass.

Here's a refined approach to achieve this:

### Step-by-Step Approach

1. **Modify `useSync.ts` to Continuously Update Spinner:**
   - Instead of only displaying the spinner toast once, update it whenever `syncStatusSubscription` changes.

2. **Update `ToastService` to Handle Prop Changes:**
   - Ensure `ToastService` can handle prop updates for toasts that are already displayed.

3. **Ensure `Toaster.vue` and `Spinner.vue` React to Prop Changes:**
   - Ensure `Spinner.vue` is reactive to prop changes.

### Detailed Implementation

#### 1. Modify `useSync.ts` for Continuous Updates

Modify `useSync.ts` to emit updates for the spinner toast whenever `syncStatusSubscription` changes.

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
      toastService.update(ToastType.Spinner, {
        allModels: newStatus.synchronizationProgress?.allModels || [],
        syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
      });
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

#### 2. Update `ToastService` to Handle Prop Changes

Add a method to `ToastService` to update existing toasts:

```typescript
public update(type: ToastType, props: Record<string, unknown>): void {
  const toasts = this.toastComponents.get(type);
  if (!toasts) return;

  this.events.emit(ToastEvent.Update, { type, props });
}
```

#### 3. Ensure `Toaster.vue` and `Spinner.vue` React to Prop Changes

Update `Toaster.vue` to handle updates:

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
  console.log('Displaying toast:', toast);
  displayedToasts.value = [...displayedToasts.value, toast];

  if (toast.timeout) {
    const timeoutHandle = window.setTimeout(() => {
      removeToast(toast.id);
    }, toast.timeout);

    toastTimeouts.value.set(toast.id, timeoutHandle);
  }
};

const updateToast = ({ type, props }): void => {
  displayedToasts.value = displayedToasts.value.map(toast => {
    if (toast.type === type) {
      return {
        ...toast,
        properties: {
          ...toast.properties,
          ...props,
        }
      };
    }
    return toast;
  });
};

const clearAllToasts = () => {
  displayedToasts.value = [];
};

onMounted(() => {
  toastService.on(ToastEvent.Display, displayToast);
  toastService.on(ToastEvent.CloseAll, clearAllToasts);
  toastService.on(ToastEvent.Update, updateToast);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Display, displayToast);
  toastService.off(ToastEvent.CloseAll, clearAllToasts);
  toastService.off(ToastEvent.Update, updateToast);
});
</script>

<style scoped>
/* styles */
</style>
```

Update `Spinner.vue` to ensure it reacts to prop changes:

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

### Summary

1. **Modify `useSync.ts`:**
   - Continuously update spinner toast props.
   
2. **Update `ToastService`:**
   - Add method to handle prop updates for existing toasts.
   
3. **Update `Toaster.vue`:**
   - Ensure it handles and passes updated props to `Spinner.vue`.

4. **Reactive Updates in `Spinner.vue`:**
   - Ensure `Spinner.vue` reacts properly to prop updates.

By ensuring continuous updates and proper reactivity, the spinner toast should actively reflect changes in `syncStatusSubscription` just like `Sync.vue`.
