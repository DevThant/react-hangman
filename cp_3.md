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
