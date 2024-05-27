To implement the requested feature, we will modify the `Spinner.vue` component to display the number of synced models out of the total models. We'll utilize the `MXSyncProgressEvent` and `MXSyncProgressStatus` to achieve this. Since these events and statuses are being used in `Sync.vue`, we can adopt a similar approach for `Spinner.vue`.

### Step-by-Step Solution

1. **Modify the `Spinner.vue` Component:**
   - Add a computed property to calculate the progress text.
   - Display the progress text in the template.

2. **Ensure `Spinner.vue` Receives the Required Data:**
   - Update the `Spinner.vue` component to accept the synchronization status as a prop.
   - Update the `toastService.displaySpinner` method to pass the synchronization status.

### Modified `Spinner.vue`

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
    <div v-if="progressText" class="progress-text">
      {{ progressText }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, defineProps, defineEmits, withDefaults } from 'vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
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
    status: undefined,
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

const progressText = computed(() => {
  if (props.status) {
    const syncedCount = props.status.syncedModels.length;
    const totalCount = props.status.allModels.length;
    return `${syncedCount}/${totalCount} models are done`;
  }
  return '';
});

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});
</script>

<style scoped>
.progress-text {
  margin-top: 8px;
  font-size: 14px;
}
</style>
```

### Modified `toastService.displaySpinner` Method

Update the `displaySpinner` method in `toastService` to pass the synchronization status.

```typescript
public displaySpinner(id: symbol, header: string, status: MXSyncProgressEvent): symbol {
  return this.displayWithId(id, ToastType.Spinner, { title: header, status });
}
```

### Usage Example in `Sync.vue`

Ensure `Sync.vue` calls `toastService.displaySpinner` with the appropriate status.

```typescript
import { toastService, ToastType } from '@/services/toast.js';

// Example of triggering the spinner with sync status
const syncStatus: MXSyncProgressEvent = {
  __typename: 'SyncProgressEvent',
  allModels: ['Model1', 'Model2', 'Model3'],
  currentModelName: 'Model2',
  syncProgressStatus: MXSyncProgressStatus.Synchronizing,
  syncedBy: 'user123',
  syncedModels: ['Model1'],
  synchronizedProduct: 'product123',
};

toastService.displaySpinner(Symbol('syncSpinner'), 'Sync in progress', syncStatus);
```

### Additional Adjustments

Ensure the `SpinnerToast` component is registered in the `toastService` if not already done.

```typescript
// In the initialization code where toasts are registered
toastService.registerToast(ToastType.Spinner, SpinnerToast);
```

This implementation ensures that the `Spinner.vue` component can display the progress of the synchronization process using the `MXSyncProgressEvent` data. Make sure to test the integration to ensure it works as expected across different clients and environments.
