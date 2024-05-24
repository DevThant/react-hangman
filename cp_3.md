The issue arises because the default value for `status` does not conform to the complete `MXSyncProgressEvent` type. Let's provide a more comprehensive default value that includes all the required properties.

Here is the corrected code for `Spinner.vue`:

```vue
<template>
  <div class="item default">
    <div class="header">
      <span class="title semi-bold">{{ titleText }}</span>
      <span class="progress">{{ syncedModels.length }}/{{ allModels.length }} models are done.</span>
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
import { computed } from 'vue';
import { onBeforeUnmount, onMounted, toRefs } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { useI18n } from 'vue-i18n';
import { ToastEvent, toastService } from '@/services/toast.js';
import { MXSyncProgressEvent, MXSyncProgressStatus } from '@ebitoolmx/gateway-types';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    status?: MXSyncProgressEvent;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    status: () => ({
      allModels: [],
      currentModelName: null,
      syncProgressStatus: MXSyncProgressStatus.Initializing,
      syncedBy: '',
      syncedModels: [],
      synchronizedProduct: ''
    })
  }
);

const { t, te } = useI18n();
const { id, title, titleVars, dismissable, status } = toRefs(props);

const dismiss = () => emit('dismiss', id.value);
const onDismiss = (dismissId: symbol) => {
  if (dismissId === id.value) {
    dismiss();
  }
};

const titleText = computed(() => (te(title.value) ? t(title.value, titleVars.value) : title.value));
const syncedModels = computed(() => status.value.syncedModels || []);
const allModels = computed(() => status.value.allModels || []);

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
});
onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});
</script>

<style scoped>
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.progress {
  font-weight: normal;
  font-size: 0.9em;
  margin-left: 10px;
}
</style>
```

This adjustment ensures that the default `status` object includes all required properties of the `MXSyncProgressEvent` type.

-----

Thanks for providing the `toastService` code. We will use it to emit synchronization events for our toaster. Let's integrate everything step-by-step.

### Step 1: Update `Spinner.vue`

We have already modified `Spinner.vue` to accept and display synchronization status data. Here is the code again for completeness:

```vue
<template>
  <div class="item default">
    <div class="header">
      <span class="title semi-bold">{{ titleText }}</span>
      <span class="progress">{{ syncedModels.length }}/{{ allModels.length }} models are done.</span>
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
import { computed } from 'vue';
import { onBeforeUnmount, onMounted, toRefs } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { useI18n } from 'vue-i18n';
import { ToastEvent, toastService } from '@/services/toast.js';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    status?: MXSyncProgressEvent;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    status: () => ({ allModels: [], syncedModels: [] })
  }
);

const { t, te } = useI18n();
const { id, title, titleVars, dismissable, status } = toRefs(props);

const dismiss = () => emit('dismiss', id.value);
const onDismiss = (dismissId: symbol) => {
  if (dismissId === id.value) {
    dismiss();
  }
};

const titleText = computed(() => (te(title.value) ? t(title.value, titleVars.value) : title.value));
const syncedModels = computed(() => status.value.syncedModels || []);
const allModels = computed(() => status.value.allModels || []);

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
});
onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});
</script>

<style scoped>
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.progress {
  font-weight: normal;
  font-size: 0.9em;
  margin-left: 10px;
}
</style>
```

### Step 2: Update `Toaster.vue`

Modify `Toaster.vue` to pass synchronization status data to `Spinner.vue`:

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
.container {
  position: absolute;
  width: 100%;
  min-height: 24px;
  display: flex;
  flex-direction: column;
  align-content: flex-end;
  z-index: 101;
  background: transparent;
  font-family: var(--font-family);
}

.container > * {
  position: relative;
  width: 100%;
  display: block;
  padding: var(--base-spacing-7);
  margin: var(--base-spacing-0);
  border-radius: var(--base-radius);
  overflow: hidden;
  z-index: 3;
  color: var(--white);
  background: var(--toaster-background-color-default);
}

.container > *.default {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-default);
}

.container > *.info {
  background: var(--toaster-background-color-info);
  border: 0;
}

.container > *.warning {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-warning);
}

.container > *.error {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-error);
}

.list-enter-active {
  transition: all 0.8s linear;
}

.list-leave-active {
  transition: all 0.4s linear;
}

.list-enter {
  opacity: 0;
}

.list-leave-to {
  opacity: 0;
}
</style>
```

### Step 3: Update `toastService` to handle synchronization events

Ensure your `toastService` is configured to emit synchronization events:

```typescript
import { DefineComponent } from 'vue';
import mitt, { Emitter, Handler } from 'mitt';
import { LoggingService } from '@ebitoolmx/logging-service/console';
import { loggingService as commonLoggingService } from './logging.js';

export enum ToastEvent {
  Display = 'display',
  Dismiss = 'dismiss',
  CloseAll = 'closeAll'
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
    const properties = props;
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

  public displayInfo(message: string, timeout?: number): symbol;

  public displayInfo(header: string, message: string, timeout?: number): symbol;

  public displayInfo(arg1: string, arg2?: string | number, arg3?: number): symbol {
    const header: string | undefined = typeof arg2 === 'string' ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === 'string' ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === 'number' ? arg2 : arg3;

    return this.display(ToastType.Simple, { title: header, text: message, type: 'info' }, timeout);
  }

  public displayWarning(message: string, timeout?: number): symbol;

  public displayWarning(header: string, message: string, timeout?: number): symbol;

  public displayWarning(arg1: string, arg2?: string | number, arg3?: number): symbol {
    const header: string

 | undefined = typeof arg2 === 'string' ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === 'string' ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === 'number' ? arg2 : arg3;

    return this.display(
      ToastType.Simple,
      { title: header, text: message, type: 'warning' },
      timeout
    );
  }

  public displayError(message: string): symbol;

  public displayError(header: string, message: string): symbol;

  public displayError(arg1: string, arg2?: string): symbol {
    const header = arg2 ? arg1 : undefined;
    const message = arg2 ?? arg1;

    return this.display(ToastType.Simple, { title: header, text: message, type: 'error' });
  }

  public displaySpinner(id: symbol, header: string, status: MXSyncProgressEvent): symbol {
    return this.displayWithId(id, ToastType.Spinner, { title: header, status });
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

  public on<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.on(type, callback);
  }

  public off<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.off(type, callback);
  }
}

export const toastService = new ToastService(commonLoggingService);
```

Now, you can use `toastService.displaySpinner(id, header, status)` to display the spinner toast with the synchronization status. For example, you can call this method whenever synchronization starts, updating the spinner toast with the progress status.


-------


To integrate the synchronization progress information into the `Spinner.vue` component, we will follow these steps:

1. **Modify `Spinner.vue` to accept synchronization status data.**
2. **Update `Toaster.vue` to pass synchronization status data to `Spinner.vue`.**
3. **Ensure the `toastService` is correctly configured to handle synchronization events.**

Here's the complete implementation:

### Step 1: Update `Spinner.vue`

Modify `Spinner.vue` to accept and display synchronization status data:

```vue
<template>
  <div class="item default">
    <div class="header">
      <span class="title semi-bold">{{ titleText }}</span>
      <span class="progress">{{ syncedModels.length }}/{{ allModels.length }} models are done.</span>
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
import { computed } from 'vue';
import { onBeforeUnmount, onMounted, toRefs } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { useI18n } from 'vue-i18n';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    status?: MXSyncProgressEvent;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    status: () => ({ allModels: [], syncedModels: [] })
  }
);

const { t, te } = useI18n();
const { id, title, titleVars, dismissable, status } = toRefs(props);

const dismiss = () => emit('dismiss', id.value);
const onDismiss = (dismissId: symbol) => {
  if (dismissId === id.value) {
    dismiss();
  }
};

const titleText = computed(() => (te(title.value) ? t(title.value, titleVars.value) : title.value));
const syncedModels = computed(() => status.value.syncedModels || []);
const allModels = computed(() => status.value.allModels || []);

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
});
onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});
</script>

<style scoped>
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.progress {
  font-weight: normal;
  font-size: 0.9em;
  margin-left: 10px;
}
</style>
```

### Step 2: Update `Toaster.vue`

Modify `Toaster.vue` to pass synchronization status data to `Spinner.vue`:

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
.container {
  position: absolute;
  width: 100%;
  min-height: 24px;
  display: flex;
  flex-direction: column;
  align-content: flex-end;
  z-index: 101;
  background: transparent;
  font-family: var(--font-family);
}

.container > * {
  position: relative;
  width: 100%;
  display: block;
  padding: var(--base-spacing-7);
  margin: var(--base-spacing-0);
  border-radius: var(--base-radius);
  overflow: hidden;
  z-index: 3;
  color: var(--white);
  background: var(--toaster-background-color-default);
}

.container > *.default {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-default);
}

.container > *.info {
  background: var(--toaster-background-color-info);
  border: 0;
}

.container > *.warning {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-warning);
}

.container > *.error {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-error);
}

.list-enter-active {
  transition: all 0.8s linear;
}

.list-leave-active {
  transition: all 0.4s linear;
}

.list-enter {
  opacity: 0;
}

.list-leave-to {
  opacity: 0;
}
</style>
```

### Step 3: Update `toastService` to handle synchronization events

Make sure your `toastService` is configured to emit synchronization events:

```typescript
// toastService.ts
import { EventEmitter } from 'events';

export enum ToastEvent {
  Display = 'display',
  Dismiss = 'dismiss',
  CloseAll = 'closeAll',
}

export const toastService = new EventEmitter();

export function displaySyncToast(status: MXSyncProgressEvent) {
  toastService.emit(ToastEvent.Display, {
    component: 'SpinnerToast',
    properties: {
      title: 'Synchronization in progress',
      status,
      timeout: 0, // keep the toast until manually dismissed
    },
  });
}
```

Now, when synchronization is in progress, you can call `displaySyncToast(status)` with the synchronization status to display the spinner toast with the progress information.

With these changes, your `Spinner.vue` component will display the synchronization progress, showing the number of synced models out of the total models.
-------

To integrate synchronization progress updates into the `spinner.vue` component using the existing WebSocket and toast services, we'll follow these steps:

1. **Update WebSocket Service**: Ensure it can handle `SyncProgress` messages.
2. **Update the composable `useSync.ts`**: Handle synchronization progress events and trigger toasts.
3. **Modify `toaster.vue` and `toast.ts`**: Ensure they can manage the new spinner toasts with progress information.
4. **Update `spinner.vue`**: Display synchronization progress.

### Step 1: Update WebSocket Service

First, make sure the WebSocket service can handle synchronization progress messages.

**src/services/webSocket.ts**:

```typescript
// Add this new topic for synchronization progress
export enum Topic {
  Error = 'error',
  ProductUpdate = 'productUpdate',
  TrackLayoutUpdate = 'updatedTrackLayout',
  ProductChanged = 'productChanged',
  ConnectUser = 'connectUser',
  ConnectedUsers = 'connectedUsers',
  DisconnectUser = 'disconnectUser',
  UpdateUser = 'updateUser',
  GetConnectedUsers = 'getConnectedUsers',
  Ping = 'ping',
  Pong = 'pong',
  Latency = 'latency',
  SyncProgress = 'syncProgress' // Add this line
}

// Ensure message handler can process SyncProgress events
private onMessage(event: MessageEvent): void {
  if (event?.data && typeof event.data === 'string') {
    try {
      const parsedData: unknown = JSON.parse(event.data);
      if (isCbssWsMessage(parsedData)) {
        switch (parsedData.topic) {
          case Topic.Ping:
            this.handlePingMessage(parsedData.message);
            break;
          case Topic.Latency:
            this.handleLatencyMessage(parsedData.message);
            break;
          case Topic.SyncProgress: // Add this case
            this.handleSyncProgressMessage(parsedData.message);
            break;
          default:
            const subscriptions = this.subscriptions.get(parsedData.topic);
            if (subscriptions) {
              subscriptions.forEach(subscriptionCallback => {
                try {
                  subscriptionCallback(parsedData);
                } catch (err) {
                  this.loggingService.error('failed to notify subscriber', { cause: err });
                }
              });
            }
        }
      } else {
        this.loggingService.warn(
          'WebSocket message received which was not in the correct format. Ignoring it.'
        );
      }
    } catch (err) {
      this.loggingService.error('WebSocket message failed', { cause: err });
    }
  } else if (event?.data) {
    this.loggingService.warn('WebSocket message event data is not text, ignoring.');
  } else {
    this.loggingService.warn('WebSocket message event is missing the data field, ignoring.');
  }
}

// Add the handleSyncProgressMessage function
private handleSyncProgressMessage(message: MXSyncProgressEvent): void {
  this.loggingService.debug('Handling SyncProgress message', message);
  eventService.emit(EventType.SyncProgress, message);
}
```

### Step 2: Update the Composable `useSync.ts`

Ensure that `useSync` handles the new progress messages and updates the toasts accordingly.

**src/composables/useSync.ts**:

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
  MXSyncProgressStatus,
  MXSyncProgressEvent // Import this
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

  const handleSyncProgress = (progress: MXSyncProgressEvent) => {
    if (syncProcessingId.value) {
      toastService.update(syncProcessingId.value, {
        progress,
        titleVars: {
          syncedBy: progress.syncedBy,
          syncedModels: progress.syncedModels.length,
          totalModels: progress.allModels.length,
        },
      });
    }
  };

  eventService.on(EventType.SyncProgress, handleSyncProgress);

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
            syncedBy: newStatus.synchronizationProgress?.syncedBy,
            syncedModels: newStatus.synchronizationProgress?.syncedModels.length,
            totalModels: newStatus.synchronizationProgress?.allModels.length,
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
    eventService.off(EventType.SyncProgress, handleSyncProgress);
  });
}
```

### Step 3: Update `toaster.vue` and `toast.ts`

Ensure the toaster component can handle and display the new spinner toasts with progress updates.

**src/services/toast.ts**:

```typescript
// Add update method to ToastService to update existing toasts
public update(id: symbol, props: Record<string, unknown>): void

 {
  this.events.emit(ToastEvent.Update, { id, props });
}

type Events = {
  [ToastEvent.Display]: Toast;
  [ToastEvent.Dismiss]: symbol;
  [ToastEvent.CloseAll]: undefined;
  [ToastEvent.Update]: { id: symbol; props: Record<string, unknown> }; // Add this line
};
```

**src/components/toaster.vue**:

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
  const toast = displayedToasts.value.find(toast => toast.id === id);
  if (toast) {
    Object.assign(toast.properties, props);
    displayedToasts.value = [...displayedToasts.value];
  }
};

const clearAllToasts = () => {
  displayedToasts.value = [];
};

onMounted(() => {
  toastService.on(ToastEvent.Display, displayToast);
  toastService.on(ToastEvent.Update, updateToast);
  toastService.on(ToastEvent.CloseAll, clearAllToasts);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Display, displayToast);
  toastService.off(ToastEvent.Update, updateToast);
  toastService.off(ToastEvent.CloseAll, clearAllToasts);
});
</script>

<style scoped>
.container {
  position: absolute;
  width: 100%;
  min-height: 24px;
  display: flex;
  flex-direction: column;
  align-content: flex-end;
  z-index: 101;
  background: transparent;
  font-family: var(--font-family);
}
.container > * {
  position: relative;
  width: 100%;
  display: block;
  padding: var(--base-spacing-7);
  margin: var(--base-spacing-0);
  border-radius: var(--base-radius);
  overflow: hidden;
  z-index: 3;
  color: var(--white);
  background: var(--toaster-background-color-default);
}
.container > *.default {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-default);
}
.container > *.info {
  background: var(--toaster-background-color-info);
  border: 0;
}
.container > *.warning {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-warning);
}
.container > *.error {
  background: var(--toaster-background-color-default);
  border: 2px solid var(--toaster-border-color-error);
}
.list-enter-active {
  transition: all 0.8s linear;
}
.list-leave-active {
  transition: all 0.4s linear;
}
.list-enter {
  opacity: 0;
}
.list-leave-to {
  opacity: 0;
}
</style>
```

### Step 4: Update `spinner.vue`

Finally, update `spinner.vue` to display the progress information.

**src/components/spinner.vue**:

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
    <div class="progress-info" v-if="syncProgress">
      {{ syncProgress.syncedModels.length }} / {{ syncProgress.allModels.length }} models are done.
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { eventService, EventType } from '@/services/event.js';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

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
const syncProgress = ref<MXSyncProgressEvent | null>(null);

const handleSyncProgressEvent = (event: MXSyncProgressEvent) => {
  syncProgress.value = event;
};

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  eventService.on(EventType.SyncProgress, handleSyncProgressEvent);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
  eventService.off(EventType.SyncProgress, handleSyncProgressEvent);
});
</script>

<style scoped>
/* styles */
</style>
```

### Summary

These changes ensure that the `spinner.vue` component will now display synchronization progress in real-time using the WebSocket service and toast notifications. This approach utilizes existing infrastructure for real-time updates and toast management, fitting seamlessly into your application's architecture.
-----------


Given that the `EventType` does not include a `SyncProgress` event, we'll need to add it to the `EventType` enum and update the `Events` type accordingly. Then we can use this new event type in the `spinner.vue` component. 

### Step 1: Update `EventType` and `Events`

Add `SyncProgress` to the `EventType` enum and update the `Events` type to include this event:

```typescript
export enum EventType {
  ScaleToFit = 'scaleToFit',
  CenterDiagram = 'centerDiagram',
  Shortcut = 'shortcut',
  Click = 'click',
  OpenDialog = 'openDialog',
  CloseDialog = 'closeDialog',
  Movement = 'movement',
  CenterSelection = 'centerSelection',
  CenterHighlighted = 'centerHighlighted',
  ResetSearch = 'resetSearch',
  GoToErrorPage = 'goToErrorPage',
  DomainObjectUpdated = 'domainObjectUpdated',
  DomainObjectDeleted = 'domainObjectDeleted',
  ToggleSidePanel = 'toggleSidePanel',
  ToggleToolsPanel = 'toggleToolsPanel',
  OpenToolsPanel = 'openToolsPanel',
  GenerateSVG = 'GenerateSVG',
  CosNodePlaced = 'CosNodePlaced',
  NotifyDetailsTable = 'NotifyDetailsTable',
  CenterIfOutOfView = 'CenterIfOutOfView',
  SyncProgress = 'syncProgress' // Add this line
}

type Events = {
  [EventType.ScaleToFit]: undefined;
  [EventType.CenterDiagram]: undefined;
  [EventType.Shortcut]: ShortcutTypes;
  [EventType.Click]: MouseEvent;
  [EventType.OpenDialog]: { dialogName: DialogNames; options?: DialogOptions };
  [EventType.CloseDialog]: undefined;
  [EventType.Movement]: Key;
  [EventType.CenterSelection]: undefined;
  [EventType.CenterHighlighted]: undefined;
  [EventType.ResetSearch]: undefined;
  [EventType.GoToErrorPage]: ErrorOptions;
  [EventType.DomainObjectUpdated]: string;
  [EventType.NotifyDetailsTable]: string;
  [EventType.DomainObjectDeleted]: string;
  [EventType.ToggleSidePanel]: { panel: Panel; options?: Record<string, any> };
  [EventType.ToggleToolsPanel]: undefined;
  [EventType.OpenToolsPanel]: undefined;
  [EventType.GenerateSVG]: undefined;
  [EventType.CosNodePlaced]: undefined;
  [EventType.CenterIfOutOfView]: undefined;
  [EventType.SyncProgress]: MXSyncProgressEvent; // Add this line
};
```

### Step 2: Update `spinner.vue`

Now that we have added the `SyncProgress` event type, we can proceed with updating `spinner.vue` to handle this event.

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
    <div class="progress-info" v-if="syncProgress">
      {{ syncProgress.syncedModels.length }} / {{ syncProgress.allModels.length }} models are done.
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { eventService, EventType } from '@/services/event.js';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

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
const syncProgress = ref<MXSyncProgressEvent | null>(null);

const handleSyncProgressEvent = (event: MXSyncProgressEvent) => {
  syncProgress.value = event;
};

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  eventService.on(EventType.SyncProgress, handleSyncProgressEvent);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
  eventService.off(EventType.SyncProgress, handleSyncProgressEvent);
});
</script>

<style scoped>
/* styles */
</style>
```

### Explanation

1. **EventType and Events Update:**
   - Added `SyncProgress` to the `EventType` enum.
   - Updated the `Events` type to include the `SyncProgress` event, mapping it to `MXSyncProgressEvent`.

2. **Template Changes:**
   - Added a `div` with class `progress-info` that conditionally displays the number of synced models out of the total models if `syncProgress` is available.

3. **Script Changes:**
   - Imported necessary services and types.
   - Defined a reactive `syncProgress` ref to store the synchronization progress event data.
   - Created `handleSyncProgressEvent` function to update `syncProgress` when a synchronization progress event is received.
   - Set up event listeners in `onMounted` to listen for synchronization progress events and clean up in `onBeforeUnmount`.

This should ensure that the `spinner.vue` component will now display the synchronization progress similarly to `sync.vue`. If there are any other adjustments needed, please let me know!

-------

To achieve the desired functionality, we need to update the `spinner.vue` component to use similar logic as `sync.vue` for displaying the synchronization progress. We'll add the logic to handle `MXSyncProgressEvent` and `MXSyncProgressStatus` in `spinner.vue` and display the number of completed synced models out of the total models.

Here are the steps:

1. **Update the `spinner.vue` template to include the synchronization progress information.**
2. **Add the necessary logic in the `script` section to handle the synchronization progress.**

Here's how you can do it:

### Step 1: Update `spinner.vue` Template

Update the template to include a section for displaying the synchronization progress:

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
    <div class="progress-info" v-if="syncProgress">
      {{ syncProgress.syncedModels.length }} / {{ syncProgress.allModels.length }} models are done.
    </div>
  </div>
</template>
```

### Step 2: Add Logic in `script` Section

Add the necessary imports and logic to handle the synchronization progress:

```vue
<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { eventService, EventType } from '@/services/event.js';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';

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
const syncProgress = ref<MXSyncProgressEvent | null>(null);

const handleSyncProgressEvent = (event: MXSyncProgressEvent) => {
  syncProgress.value = event;
};

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  eventService.on(EventType.SyncProgress, handleSyncProgressEvent);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
  eventService.off(EventType.SyncProgress, handleSyncProgressEvent);
});
</script>
```

### Explanation

1. **Template Changes:**
   - Added a `div` with class `progress-info` that conditionally displays the number of synced models out of the total models if `syncProgress` is available.

2. **Script Changes:**
   - Imported necessary services and types.
   - Defined a reactive `syncProgress` ref to store the synchronization progress event data.
   - Created `handleSyncProgressEvent` function to update `syncProgress` when a synchronization progress event is received.
   - Set up event listeners in `onMounted` to listen for synchronization progress events and clean up in `onBeforeUnmount`.

This implementation ensures that the `spinner.vue` component will now display the synchronization progress similarly to `sync.vue`. If you need any further adjustments or have specific requirements, feel free to let me know!
