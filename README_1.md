I have a vue 3 app, written in typescript and pinia, and java as a backend but we do not need to concern with the backend. Everything that we have to work on is the frontend.


So the app is something like an autocad that engineers can draw stuffs on, it support collaborations so multiple users can work on one project at the same time. And right now I'm trying to upgrade certain components to improve the experience of the collaboration inside the app.

I have this toaster feature, it supports multiple toaster type, but the one that I want to focus on right now is called the sub toaster type the spinner.

So before going to the spinner you have to know the other use case related to the toaster as well. This is necessary and important.

So if a person is working on the project, they can do something called "sync", this is used to sync project across every users screen. So if multiple people is actively working on the project, when a person clicks on the sync, it trigger the syncrhonisation process, locking everyone from editing and sync the project for everyone. 

During this process, the person who trigger the sync (who clicks on the sync button) get a pop up screen, which is a Sync.vue component. Displaying synchronisation message and all the models inside the program, and it is a live process, syncing one models after another and once a model is finished, the color of the model changes and the done icon is displayed. But only person who can see this sync window/popup is the one who triggers it, the rest of the collaborators, working on the same program, just only receive a toaster and they cannot edit the program until the sync process is complete, they don't get the popup window of the sync (Sync.vue). The toaster the rest of the collaborator receive is the Spinner.vue Toast, which display the message that sync is in progress and who triggers it. 

Okay now the main point, I'm trying to improve the message of Spinner.vue to be more informative. Currently it only shows "Synchronisation has been requested by [user]" and a lodaing indicator beneath it. I want to include the progress of the sync in this toaster, specifically how many models has been synced from total. I want to display this message between the existing message  and the loading, "3/7 models are done.".

As for getting the total models and how many models has been synced, we probably can retrieve these information directly from the Sync.vue component which is actively doing the syncing process. We actively and reactively get the live data from the Sync.vue and display it directly inside the Spinner.vue, updating the toaster live with the models sync process.


------
I've already tried to improve this feature by creating a new common store to be used between the Spinner.vue and the Sync.vue, which manages two values, totalModels and syncedModels. And it has been linked up with both Spinner.vue and the Sync.vue, retrieving data from sync.vue and delivering them to the spinner.vue.

But I'm encountering one huge problem, the implementation is working but not quite the way I want and have a big problem. 

So after observing the functionality, I found the weird and janky interactions between the spinner and sync.

Lets say person A, B and C are working together on the project.

The condition is that none of the people wokring on the project has triggered the sync yet. Then person A trigger the sync, he gets the popup window of sync, working well, sync is done after a while after completing all the models. BUT, this is whats wrong with current implementation, the other collaborators B and C get the toaster, but the toaster says 0/0 models are done. Which is because I've set the initial value of the synced and total in the common store to be 0. They're not updating live.

let me continued, so A synced, B and C get 0/0 spinner toaster. Now B try to sync and A get the 7/7 models are done which is correct in term of the total numbers and total synced models, but not live, and the sycnedModels does not start from 0 to 1 to 2 to and so on to 7. It is 7/7 when the toaster appear instead of 0/0 like B and C. I'm not sure what is the reason, or the implementation is even working, also have a hunch that the toaster get updated if person sync before and its something like keeping instance of their own sync rather than getting the live data when other people trigger the sync.



So these are the codes.

Sync.vue
//src/components/common/dialog/Sync.vue


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
import { useSyncProgressStore } from '@/stores/syncProgressState'; // Import the shared state
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
const syncProgressStore = useSyncProgressStore();
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
  // Update shared state with initial progress data
  const totalModels = props.status?.allModels.length || 3;
  const syncedModels = props.status?.syncedModels.length || 3;
  syncProgressStore.setTotalModels(totalModels);
  syncProgressStore.setSyncedModels(syncedModels);
  console.log(`${props.status}`);
  console.log(`Synced Models : ${syncedModels}`);
  console.log(`Total Models : ${totalModels}`);
}
watch(
  () => props.status,
  newStatus => {
    if (newStatus) {
      // Update shared state whenever the status changes
      syncProgressStore.setTotalModels(newStatus.allModels.length);
      syncProgressStore.setSyncedModels(newStatus.syncedModels.length);
    }
  },
  { immediate: true, deep: true }
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
      margin-right: var (--base-spacing-1);
    }
  }
  .syncing {
    color: var (--font-color);
    fill: var (--font-color);
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
      color: var (--link-color);
    }
    p {
      padding-right: var (--base-spacing-1);
    }
  }
  .failed-model {
    display: flex;
    align-items: center;
    color: var (--danger);
    fill: var (--danger);
    p {
      padding-left: var (--base-spacing-1);
    }
  }
}
</style>


Spinner.vue
//src/components/common/toast/Spinner.vue

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
    <p class="progress-text">{{ progressText }}</p>
    <!-- New line for progress text -->
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>
<script setup lang="ts">
import { onBeforeUnmount, onMounted, computed } from 'vue';
import { useSyncProgressStore } from '@/stores/syncProgressState'; // Import the shared state
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
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false
  }
);
const emit = defineEmits<{ dismiss: [id: symbol] }>();
const { t, te } = useI18n();
const syncProgressStore = useSyncProgressStore();
const dismiss = () => emit('dismiss', props.id);
const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};
const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;
const progressText = computed(
  () => `${syncProgressStore.synced} / ${syncProgressStore.total} models are done`
); // Computed property for progress text
onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
});
onBeforeUnmount(() => {
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
.header .dismiss:hover {
  fill: var(--toaster-dismiss-icon);
}
.spinner {
  margin: 15px 0 15px;
}
</style>


syncProgressState.ts

//src/stores/syncProgressState.ts


import { acceptHMRUpdate, defineStore } from 'pinia';
import { ref, computed } from 'vue';
export const useSyncProgressStore = defineStore('syncProgress', () => {
  const totalModels = ref<number>(0);
  const syncedModels = ref<number>(0);
  // Getters
  const total = computed(() => totalModels.value);
  const synced = computed(() => syncedModels.value);
  // Actions
  const setTotalModels = (total: number) => {
    totalModels.value = total;
  };
  const setSyncedModels = (synced: number) => {
    syncedModels.value = synced;
  };
  return {
    total,
    synced,
    setTotalModels,
    setSyncedModels
  };
});
if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useSyncProgressStore, import.meta.hot));
}

Below are the optionals but maybe they'll provide useful information.

Toaster.vue
//src/components/common/toaster/Toaster.vue



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

Toast.ts
//src/services/toast.ts



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
/**
 * All the required details for displaying a toast in the Toaster component.
 */
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
/**
 * Handles toaster events and components for the toaster component.
 */
export class ToastService {
  protected toastComponents = new Map<ToastType | string, DefineComponent>();
  protected events: Emitter<Events> = mitt<Events>();
  public constructor(protected loggingService: LoggingService) {}
  /**
   * Registers a toast into the service so that it can be referenced later when displaying
   */
  public registerToast(type: ToastType | string, toast: any) {
    this.toastComponents.set(type, toast);
  }
  /**
   * Displays a toast with a given id and of specified type. This will emit an event to the Toaster
   * component with the necessary details to display a toast. This is the most low-level endpoint
   * that all details to be able to display a toast.
   */
  public displayWithId(
    id: symbol,
    type: ToastType | string,
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
  /**
   * Displays a toast of a specified type. This will auto generate an id and return it before
   * emitting an event to the Toaster component. See displayWithId().
   */
  public display(
    type: ToastType | string,
    props: Record<string, unknown> = {},
    timeout: number | undefined = undefined
  ): symbol {
    return this.displayWithId(Symbol('toast'), type, props, timeout);
  }
  /**
   * Displays a simple info toast with a provided message and optional timeout
   */
  public displayInfo(message: string, timeout?: number): symbol;
  /**
   * Displays a simple info toast with a provided header, message, and optional timeout
   */
  public displayInfo(header: string, message: string, timeout?: number): symbol;
  /**
   * A fallback implementation for displayInfo that will determine which argument is the toast
   * message, header, and timeout based on the argument types.
   */
  public displayInfo(arg1: string, arg2?: string | number, arg3?: number): symbol {
    const header: string | undefined = typeof arg2 === 'string' ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === 'string' ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === 'number' ? arg2 : arg3;
    return this.display(ToastType.Simple, { title: header, text: message, type: 'info' }, timeout);
  }
  /**
   * Displays a simple warning toast with a provided message and optional timeout
   */
  public displayWarning(message: string, timeout?: number): symbol;
  /**
   * Displays a simple warning toast with a provided header, message, and optional timeout
   */
  public displayWarning(header: string, message: string, timeout?: number): symbol;
  /**
   * A fallback implementation for displayWarning that will determine which argument is the toast
   * message, header, and timeout based on the argument types.
   */
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
  /**
   * Displays a simple error toast with a provided message
   */
  public displayError(message: string): symbol;
  /**
   * Displays a simple error toast with a provided message and header
   */
  public displayError(header: string, message: string): symbol;
  /**
   * A fallback implementation for typing.
   */
  public displayError(arg1: string, arg2?: string): symbol {
    const header = arg2 ? arg1 : undefined;
    const message = arg2 ?? arg1;
    return this.display(ToastType.Simple, { title: header, text: message, type: 'error' });
  }
  /**
   * Will display a spinner toaster with the provided id and header.
   */
  public displaySpinner(id: symbol, header: string): symbol {
    return this.displayWithId(id, ToastType.Spinner, { title: header });
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
  /**
   * Subscribe to the closeAll toast event in the toast service
   */
  public on<Type extends ToastEvent.CloseAll>(type: Type, callback: Function): void;
  /**
   * Subscribe to the dismiss toast event in the toast service
   */
  public on<Type extends ToastEvent.Dismiss>(type: Type, callback: Function): void;
  /**
   * Subscribe to the display toast event. This event will pass information about the toast to be
   * displayed.
   */
  public on<Type extends ToastEvent.Display>(type: Type, callback: (toast: Toast) => void): void;
  /**
   * A fallback type for subscribing to events in the toast service.
   */
  public on<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.on(type, callback);
  }
  /**
   * Unsubscribe from from an event in the toast service
   */
  public off<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.off(type, callback);
  }
}
export const toastService = new ToastService(commonLoggingService);
