Thank you for providing the `event.ts` service file. Based on this file, it is clear that there is no `EventType.Sync`. Instead, we will use the same approach that `Sync.vue` uses to handle synchronization events.

Since `Sync.vue` directly handles synchronization data, we will need to find the method used in `Sync.vue` for subscribing to synchronization updates. Then, we will replicate that method in `Spinner.vue`.

### Step-by-Step Implementation

1. **Identify Subscription Method in `Sync.vue`**: Review the subscription method for synchronization events.
2. **Apply Similar Logic to `Spinner.vue`**: Implement the same method in `Spinner.vue` to handle synchronization updates.

### Step 1: Identify Subscription Method in `Sync.vue`

From your provided `Sync.vue`, it uses stores and services to manage synchronization states. Let's identify the key parts handling synchronization updates.

### Step 2: Apply Similar Logic to `Spinner.vue`

We will now apply similar logic in `Spinner.vue` to handle synchronization updates. Here's how we can update `Spinner.vue`:

### Updated `Spinner.vue`

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
    <div v-if="status" class="sync-status">
      {{ status.syncedModels.length }}/{{ status.allModels.length }} models are done.
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { MXSyncProgressEvent } from '@ebitoolmx/gateway-types';
import { eventService, EventType } from '@/services/event.js';
import { useAppStore } from '@/stores/app.js';
import { useProductsStore } from '@/stores/products.js';
import { useConnectionsStore } from '@/stores/connections.js';

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

const status = ref<MXSyncProgressEvent | null>(null);

const appStore = useAppStore();
const productsStore = useProductsStore();
const connectionsStore = useConnectionsStore();

function handleSyncProgress(event: MXSyncProgressEvent) {
  status.value = event;
}

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
  eventService.on(EventType.DomainObjectUpdated, handleSyncProgress);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
  eventService.off(EventType.DomainObjectUpdated, handleSyncProgress);
});
</script>

<style scoped>
.sync-status {
  margin-top: 10px;
  font-weight: bold;
}
</style>
```

### Explanation

- **Subscription to Sync Events**: Subscribed to `EventType.DomainObjectUpdated` which seems to be an appropriate existing event type.
- **Handling Synchronization Data**: The `handleSyncProgress` function updates the `status` ref with the synchronization data.
- **Display Synchronization Progress**: The template displays the progress, similar to `Sync.vue`.

### Verifying Integration

Make sure your synchronization events are correctly emitting `EventType.DomainObjectUpdated` or any other appropriate event that `Spinner.vue` can listen to. If the event type needs to be different, ensure you match the event type used for synchronization updates.

Let me know if this approach aligns with your setup or if there are any further adjustments needed.
