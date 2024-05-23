To implement the required feature of displaying the sync progress in the Spinner.vue component, we need to follow these steps:

1. **Enhance Sync.vue to emit progress updates**: Sync.vue will emit events with the progress data.
2. **Listen to progress updates in Spinner.vue**: Update Spinner.vue to listen to these events and display the progress.

Let's go through these steps in detail:

### Step 1: Enhance Sync.vue to Emit Progress Updates

We will emit events from Sync.vue with the current progress of the sync process.

```vue
<template>
  <app-dialog-container title-key="sync.modal.title" data-testid="dialog-sync">
    <!-- ... existing code ... -->
  </app-dialog-container>
</template>

<script setup lang="ts">
import { watch, computed } from "vue";
// ... other imports ...

// Define props and other variables as before

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType,
  });

  emitProgress();
}

function emitProgress() {
  if (props.status) {
    const progress = {
      syncedModels: props.status.syncedModels.length,
      totalModels: props.status.allModels.length,
    };
    eventService.emit("syncProgress", progress);
  }
}

// Watcher to emit progress when the status changes
watch(
  () => props.status,
  (newStatus) => {
    emitProgress();
  },
  { deep: true }
);

// Existing code for watches, computed properties, and functions

</script>

<style scoped>
/* Existing styles */
</style>
```

### Step 2: Update Spinner.vue to Listen to Progress Updates

We'll modify Spinner.vue to listen to these progress events and update its display accordingly.

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
    <div class="progress">
      <span>{{ progressText }}</span>
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
import { eventService } from "@/services/event.js";

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
const titleText: string = te(props.title)
  ? t(props.title, props.titleVars)
  : props.title;
const progressText = ref("Synchronisation has been requested by user@example.com");

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const updateProgressText = (progress) => {
  progressText.value = `Synchronisation has been requested by user@example.com\n${progress.syncedModels}/${progress.totalModels} models are done.`;
};

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  eventService.on("syncProgress", updateProgressText);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
  eventService.off("syncProgress", updateProgressText);
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

.progress {
  margin: 10px 0;
}

.spinner {
  margin: 15px 0 15px;
}
</style>
```

With these changes, the Spinner.vue component will now display the sync progress, showing the number of models that have been synced out of the total models, in real time.

### Next Steps

**a.** Add unit tests to ensure the progress update mechanism works correctly.
**b.** Optimize the event emission to avoid unnecessary updates.

Would you like to proceed with these steps or have any further modifications in mind?
