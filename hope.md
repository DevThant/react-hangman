### Analysis and Interaction Summary

#### **Editor.vue (InfrastructureEditor.vue)**

1. **GoJS Diagram Setup**:
    - The GoJS diagram is set up with `useInjectDiagram(cbssDiagramKey)`.
    - It initializes and updates the diagram based on interactions (`onDiagramMounted`, `onModelChanged`, etc.).

2. **Highlight and Selection**:
    - Functions like `clearTrackItemAdornments`, `updateTrackItemAdornments`, `onSelectionChanged`, and `onHighlightedChanged` handle the highlighting and selection changes in the diagram.

3. **Event Handling**:
    - It listens to various events (`LinkDrawn`, `LinkReshaped`, `SelectionMoved`, `PartResized`) to update the state.

4. **Store Interaction**:
    - Utilizes `infrastructureStore`, `editorStore`, `productsStore`, etc., for managing state and performing actions such as deleting, adding, or updating domain objects.

#### **TrackItemProperty.vue**

1. **Track Item Management**:
    - Handles adding, deleting, and highlighting track items through methods like `addValue`, `deleteValue`, and `selectItemGroup`.

2. **Highlighting and Selection**:
    - Uses `eventService` to emit events for highlighting (`EventType.CenterHighlighted`).

3. **Computed Properties**:
    - `currentIds`, `groupedTrackItems`, `isEditing` manage and track the state of track items.

#### **infrastructure.ts**

1. **State Management**:
    - Manages state for tracks, track items, nodes, detectors, signal equipment rooms, position corrections, notes, lines, item groups, area groups, main tracks, line profiles, site, and model metadata.

2. **Getters and Actions**:
    - Provides getters for retrieving domain objects, track items, nodes, lines, etc.
    - Includes actions for adding, deleting, and updating domain objects, subscribing to updates, and handling WebSocket messages.

3. **Events**:
    - Utilizes `eventService` to emit and listen for events such as `DomainObjectUpdated` and `DomainObjectDeleted`.

#### **editor.ts**

1. **State Management**:
    - Manages editor state including selected and highlighted items, editor mode, side panels, split views, etc.

2. **Actions**:
    - Provides actions for setting selected and highlighted items, toggling split views, and handling domain object deletions.

3. **Events**:
    - Tracks events such as editor mode changes and split view changes using `metricsService`.

#### **event.js**

1. **Event Service**:
    - Implements a global event bus (`mitt`) for emitting and listening to events anywhere in the app.

2. **Event Types**:
    - Defines various event types (`ScaleToFit`, `CenterDiagram`, `Shortcut`, `OpenDialog`, etc.).

### Connection and Interaction

- **Highlighting and Selection**:
    - When a track item is highlighted or selected in `TrackItemProperty.vue`, it emits events (`EventType.CenterHighlighted`) using `eventService`.
    - `Editor.vue` listens to these events and updates the GoJS diagram accordingly.

- **State Management**:
    - Both `Editor.vue` and `TrackItemProperty.vue` interact with the Vuex store (`infrastructureStore`, `editorStore`) to manage the state of track items and their highlighting/selection.

- **Event Handling**:
    - `eventService` is used for global communication between components, ensuring that actions in one component (e.g., deleting a track item) are reflected in others.

### Next Steps

To implement the desired feature of removing a track item from `TrackItemProperty.vue` without deleting it from the GoJS diagram, follow these steps:

1. **Update `TrackItemProperty.vue`**:
    - Modify the `deleteValue` method to emit an event or update the store to remove the track item from the `TrackItemProperty` list but not from the diagram.

2. **Handle Removal in `Editor.vue`**:
    - Listen for the event emitted by `TrackItemProperty.vue` in `Editor.vue` and update the GoJS diagram to reflect the removal of the track item from the selection/highlight list.

### Implementation

#### **TrackItemProperty.vue**

Update the `deleteValue` method to emit an event for removal:

```javascript
const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  
  // Emit an event to remove the track item from the property list
  eventService.emit(EventType.RemoveTrackItem, excludeXmiId);
};
```

#### **Editor.vue**

Listen for the `RemoveTrackItem` event and update the diagram:

```javascript
onMounted(() => {
  eventService.on(EventType.RemoveTrackItem, (xmiId: string) => {
    const trackItem = infrastructureStore.getTrackItem(xmiId);
    if (trackItem) {
      // Remove the track item from the selection/highlight list
      editorStore.setHighlighted(
        new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
      );
      
      const newSelection = editorStore.selected.ids.filter(id => id !== xmiId);
      editorStore.setSelected(new PlainSelection(newSelection));
      
      // Update the track item adornments
      updateTrackItemAdornments();
    }
  });
  
  // Existing event listeners...
  eventService.on(EventType.Shortcut, shortcutHandler);
  gojsDiagram.diagram.commandHandler.doKeyDown = onDiagramKeyDown;
  gojsDiagram.diagram.addModelChangedListener(onModelChanged);
  gojsDiagram.diagram.addDiagramListener('LinkDrawn', onLinkedOrRelinked);
  gojsDiagram.diagram.addDiagramListener('LinkReshaped', onLinkReshaped);
  gojsDiagram.diagram.addDiagramListener('SelectionMoved', onSelectionMoved);
  gojsDiagram.diagram.addDiagramListener('PartResized', onPartResized);
});
```

### Next Steps for Further Investigation

1. **Test the Changes**:
    - Test the updated functionality to ensure that track items are correctly removed from the `TrackItemProperty` list without being deleted from the GoJS diagram.

2. **Additional Features**:
    - Consider adding features like undo/redo for track item removal or providing a confirmation dialog before removal.

**Suggestions for the next user turn**:

**a.** Run the updated Vue components in the development environment to test the new feature.

**b.** Add unit tests for the `deleteValue` method in `TrackItemProperty.vue` and the event handling in `Editor.vue`.

-------------
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

------

To ensure that the spinner toast actively reflects changes in synchronization status, similar to `Sync.vue`, we need to ensure that the toast updates its state dynamically in response to changes in the synchronization progress. This means the toast should be able to react to multiple state updates and not just a one-time initialization.

### Steps to Ensure Active Reflection of Changes

1. **Emit Updates Continuously:**
   Ensure that updates to the synchronization status are continuously emitted and handled by the `ToastService`.

2. **Ensure ToastService Can Handle Updates:**
   Make sure `ToastService` can handle multiple updates for the same toast instance.

3. **Ensure Toaster.vue and Spinner.vue React to Updates:**
   Ensure `Toaster.vue` and `Spinner.vue` are set up to reactively update when they receive new props.

### Detailed Steps

1. **Emit Updates Continuously in `useSync.ts`:**
   
   Make sure you emit toast updates whenever there is a change in synchronization status, not just once.

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

       console.log('Sync Status Updated:', newStatus.synchronizationProgress);

       if (syncProcessing || syncInitialising) {
         toastService.update(syncProcessingId.value, {
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

2. **Extend `ToastService` to Handle Updates:**
   
   Modify the `ToastService` to support updates to existing toasts.

   ```typescript
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

     public update(id: symbol, props: Record<string, unknown> = {}) {
       const toast = this.toastComponents.get(ToastType.Spinner);
       if (!toast) return;

       this.events.emit(ToastEvent.Display, { id, component: toast, properties: props });
       this.loggingService.debug(`Updating toast of type '${ToastType.Spinner}'`, props);
     }

     public display(
       type: ToastType,
       props: Record<string, unknown> = {},
       timeout: number | undefined = undefined
     ): symbol {
       return this.displayWithId(Symbol('toast'), type, props, timeout);
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

3. **Ensure `Toaster.vue` Handles Updates Correctly:**

   Ensure `Toaster.vue` listens for updates and passes them to `Spinner.vue`.

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
     const existingToastIndex = displayedToasts.value.findIndex(t => t.id === toast.id);
     if (existingToastIndex >= 0) {
       displayedToasts.value[existingToastIndex] = { ...displayedToasts.value[existingToastIndex], ...toast };
       displayedToasts.value = [...displayedToasts.value]; // force reactivity
     } else {
       displayedToasts.value = [...displayedToasts.value, toast];
     }

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

4. **Ensure `Spinner.vue` Reactively Updates:**

   Make sure `Spinner.vue` is set up to reactively update when props change.

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
       <div v-if="allModels.length || syncedModels.length">
         <p>All Models: {{ allModels.join(', ') }}</p>
         <p>Synced Models: {{ syncedModels.join(', ') }}</p>
       </div>
     </div>
   </template>

   <script setup lang="ts">
   import { onBeforeUnmount, onMounted, watch } from 'vue';
   import AppIconButton from '@/components/common/icon/IconButton.vue';
   import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
   import

 { ToastEvent, toastService } from '@/services/toast.js';
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

By ensuring the `ToastService` can handle updates and `Toaster.vue` and `Spinner.vue` reactively update with the new data, you should achieve the desired active reflection of synchronization status in the spinner toast.
