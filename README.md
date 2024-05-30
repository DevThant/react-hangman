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

              <div v-if="status?.currentModelName && isSyncUnsuccessfulDialog" class="failed-model">
                <app-icon name="sync_error" />
                <p class="lead dialog-content-message">{{ status.currentModelName }}</p>
              </div>

              <div v-if="status?.currentModelName && isSyncFailedDialog" class="failed-initialized">
                <p class="lead dialog-content-message">
                  {{ t('sync.modal.message.modelFailed', { modelName: status.currentModelName }) }}
                </p>
              </div>

              <div class="help-desk">
                <p class="lead dialog-content-message">
                  {{ t('sync.modal.message.contact') }}
                  <a
                    :href="t('common.link')"
                    rel="noopener noreferrer"
                    target="_blank"
                    class="lead dialog-content-message"
                  >
                    {{ t('common.helpSupport') }}
                  </a>
                </p>
              </div>
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
      return 'sync.modal.message.modelFailed';
    case SyncDialogType.Unsuccessful:
      return 'sync.modal.message.unsuccessful';
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
}

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
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
    fill: var(--success);
  }
}

.unsuccessful {
  display: flex;
  flex-direction: column;
  align-items: center;

  .unsuccessful-message {
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }

  .failed-initialized {
    display: block;
    align-items: center;
    p {
      padding-left: var(--base-spacing-1);
    }
  }

  .help-desk {
    display: flex;
    align-items: center;
    margin-top: var(--base-spacing-2);

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }
}
</style>
```

### Explanation:
1. **Template Changes**: The contact help desk message and link are now wrapped in a single `<div class="help-desk">` container to ensure they stay together.
2. **CSS Adjustments**: The `.help-desk` class is styled to display the content inline and ensure proper alignment and spacing.

This should ensure that the contact message and the help desk link appear together on the same line.




-

-----------
 address the issue of ensuring the failed model message and the contact help desk link are displayed on separate lines, you can adjust the CSS and HTML structure slightly. Here's how you can modify your Sync.vue file to achieve this:

1. **Update the Sync.vue Template**:
   - Make sure the elements are placed correctly to respect the layout.

2. **Update the CSS**:
   - Ensure the CSS styles are correctly applied to create a proper layout for the messages.

Here is the updated version of your Sync.vue file with these adjustments:

### Sync.vue

```vue
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

              <div v-if="status?.currentModelName && isSyncUnsuccessfulDialog" class="failed-model">
                <app-icon name="sync_error" />
                <p class="lead dialog-content-message">{{ status.currentModelName }}</p>
              </div>

              <div v-if="status?.currentModelName && isSyncFailedDialog" class="failed-initialized">
                <p class="lead dialog-content-message">
                  {{ t('sync.modal.message.modelFailed', { modelName: status.currentModelName }) }}
                </p>
              </div>

              <div class="help-desk">
                <p class="lead dialog-content-message">{{ t('sync.modal.message.contact') }}</p>
                <a
                  :href="t('common.link')"
                  rel="noopener noreferrer"
                  target="_blank"
                  class="lead dialog-content-message"
                >
                  {{ t('common.helpSupport') }}
                </a>
              </div>
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
      return 'sync.modal.message.modelFailed';
    case SyncDialogType.Unsuccessful:
      return 'sync.modal.message.unsuccessful';
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
}

watch(
  () => connectionsStore.isOffline,
  nowOffline => {
    if (nowOffline)
      eventService.emit(EventType.OpenDialog, {
        dialogName: DialogNames.Sync,
        options: {
          props: {
            dialogType: SyncDialog

Type.Offline
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
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
    fill: var(--success);
  }
}

.unsuccessful {
  display: flex;
  flex-direction: column;
  align-items: center;

  .unsuccessful-message {
    display: flex;
    flex-direction: column;
    align-items: center;
    text-align: center;

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }

  .failed-initialized {
    display: block;
    align-items: center;
    p {
      padding-left: var(--base-spacing-1);
    }
  }

  .help-desk {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: var(--base-spacing-2);

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }
}
</style>
```

This code makes sure the failed model message and the contact help desk link are displayed on separate lines and are properly aligned. The `.help-desk` class ensures the help desk information is separated and displayed as a block below the failed model message.


------

To address the issue of incorporating more detailed information into the initialization failed dialog, we can update the `Sync.vue` component and the i18n localization file. Here’s how you can do it:

### Update `Sync.vue`

Add the necessary condition to handle the detailed initialization failure message.

```vue
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
        :message-line1="detailedMessage ? t(detailedMessage) : message ? t(message) : message"
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
            <div v-if="status?.currentModelName && isSyncFailedDialog" class="failed-model">
              <app-icon name="sync_error" />
              <p class="lead dialog-content-message">{{ t('sync.modal.message.modelFailed', { modelName: status.currentModelName }) }}</p>
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
const detailedMessage = computed<PickupPaths<LocaleMessage> | null>(() => {
  if (props.dialogType === SyncDialogType.InitialisingFailed) {
    return props.status?.currentModelName ? 'sync.modal.message.modelFailed' : null;
  }
  return null;
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
}
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
/* styles */
</style>
```

### Update the Localization File

Add a new entry for the detailed message when

 the initialization fails.

```typescript
export const en = {
  sync: {
    modal: {
      title: 'Synchronization',
      header: {
        requested: 'Synchronization Requested',
        initialising: 'Initialization',
        processing: 'Synchronization in Progress',
        successful: 'Synchronization Successful',
        initialisingFailed: 'Initialization Failed',
        unsuccessful: 'Synchronization Unsuccessful',
        offline: 'Offline'
      },
      message: {
        requested: 'Synchronization has been requested.',
        progress: 'Synchronization is in progress...',
        offline: 'You are offline.',
        unsuccessful: 'Synchronization was unsuccessful.',
        contact: 'Please contact the help desk.',
        modelFailed: 'The {modelName} model couldn’t be initialized. If the issue remains, contact the help desk.'
      }
    },
    toaster: {
      requested: 'Synchronization',
      progress: 'Synchronization has been requested by {syncedBy}\n{done}/{total} models are done.',
      successful: 'Synchronization Successful',
      unsuccessful: 'Synchronization Unsuccessful'
    }
  }
};
```

By making these adjustments, the dialog will now display a more informative message about the specific model that failed during initialization, matching the desired output in the example provided. This approach keeps the implementation generic and easily extendable for future enhancements.

-----

It looks like the issue is with how the `messageText` is computed or rendered. Let's ensure that the `messageText` is correctly computed using the `message` and `messageVars` props. Here's how to make sure that happens:

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
    <div class="content">
      <p>{{ messageText }}</p>
    </div>
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, computed } from 'vue';
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
    message?: string;
    messageVars?: Record<string, unknown>;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    message: '',
    messageVars: () => ({})
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

const messageText = computed(() => {
  return te(props.message) ? t(props.message, props.messageVars) : props.message;
});

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
```

### Verify `useSync.ts`

Ensure that the `toastService.update` is being called correctly with the new message and messageVars:

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
            message: 'sync.toaster.progress',
            messageVars: {
              done: newStatus.synchronizationProgress?.syncedModels.length || 0,
              total: newStatus.synchronizationProgress?.allModels.length || 0
            }
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
        } else if (syncProcessing) {
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
        } else if (syncSuccessful) {
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
        } else if (syncInitialisationFailed) {
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
        } else if (syncUnsuccessful) {
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
        }
      } else if ((syncProcessing || syncInitialising) && !syncProcessingId.value) {
        console.log('new status', newStatus.synchronizationProgress);
        syncProcessingId.value = toastService.display(ToastType.Spinner, {
          type: 'default',
          title: 'sync.toaster.requested',
          titleVars: {
            syncedBy: newStatus.synchronizationProgress?.syncedBy
          },
          message: 'sync.toaster.progress',
          messageVars: {
            done: newStatus.synchronizationProgress?.syncedModels.length || 0,
            total: newStatus.synchronizationProgress?.allModels.length || 0
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
  });
}
```

### Verify `i18n` Setup

Ensure your `i18n` setup includes the `sync.toaster.progress` key with proper placeholders for `done` and `total`.

#### `locales/en.js`

```javascript
export default {
  common: {
    dismiss: 'Dismiss'
  },
  sync: {
    toaster: {
      requested: 'Synchronization has been requested by {syncedBy}',
      progress: '{done}/{total} models are done',
      successful: 'Synchronization successful',
      unsuccessful: 'Synchronization unsuccessful'
    }
  }
};
```

By ensuring that the `messageText` is computed using the `message` and `messageVars` props, it should correctly update and display the synchronization progress.


------
To ensure the toast updates correctly with the synchronization progress, let's go through the necessary steps to verify and fix the implementation. 

### Ensure `ToastEvent.Update` Handling in `Toaster.vue`

First, let's make sure that the `Toaster.vue` is correctly handling the `ToastEvent.Update` event.

#### `Toaster.vue`

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

### Ensure `ToastEvent.Update` Handling in `toast.ts`

Next, let's verify the `toast.ts` file is correctly handling the `ToastEvent.Update` event.

#### `toast.ts`

```typescript
import { DefineComponent } from 'vue';
import mitt, { Emitter, Handler } from 'mitt';
import { LoggingService } from '@ebitoolmx/logging-service/console';
import { loggingService as commonLoggingService } from './logging.js';

export enum ToastEvent {
  Display = 'display',
  Dismiss = 'dismiss',
  CloseAll = 'closeAll',
  Update = 'update'
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
  [ToastEvent.Update]: { id: symbol; props: Record<string, unknown> };
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
    this.loggingService.debug(`Displaying toast of type '${type}'`);

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
   * Updates a toast with a given id and new properties. This will emit an update event to the
   * Toaster component to update the toast's properties.
   */
  public update(id: symbol, props: Record<string, unknown>): void {
    this.events.emit(ToastEvent.Update, { id, props });
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
   * Subscribe to the update toast event. This event will pass information about the toast to be
   * updated.
   */
  public on<Type extends ToastEvent.Update>(type: Type, callback: (update: { id: symbol; props: Record<string, unknown> }) => void): void;

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
```

### Ensure `useSync.ts` Correctly Calls `toastService.update`

Finally, ensure that `useSync.ts` correctly calls the `toastService.update` function to update the toast properties.

#### `useSync.ts`

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
            message: 'sync.toaster.progress',
            messageVars: {
              done: newStatus.synchronizationProgress?.syncedModels.length || 0,
              total: newStatus.synchronizationProgress?.allModels.length || 0
            }
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
        } else if (syncProcessing) {
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
        } else if (syncSuccessful) {
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
        } else if (syncInitialisationFailed) {
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
        } else if (syncUnsuccessful) {
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
        }
      } else if ((syncProcessing || syncInitialising) && !syncProcessingId.value) {
        console.log('new status', newStatus.synchronizationProgress);
        syncProcessingId.value = toastService.display(ToastType.Spinner, {
          type: 'default',
          title: 'sync.toaster.requested',
          titleVars: {
            syncedBy: newStatus.synchronizationProgress?.syncedBy
          },
          message: 'sync.toaster.progress',
          messageVars: {
            done: newStatus.synchronizationProgress?.syncedModels.length || 0,
            total: newStatus.synchronizationProgress?.allModels.length || 0
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
  });
}
```

### Summary

- Ensure the `Toaster.vue` component listens for `ToastEvent.Update` and correctly updates the toast properties.
- Ensure the `toast.ts` file emits `ToastEvent.Update` with the correct properties.
- Ensure `useSync.ts` calls `toastService.update` with the correct toast ID and properties.

With these steps, the spinner toast should correctly update its message and other properties based on the synchronization progress.


-----
Absolutely, we can make the `SpinnerToast` component more generic by passing a `message` prop instead of the sync-specific properties. This way, the component won't have any sync-specific logic and can be used for any type of spinner toast.

### Updated `SpinnerToast.vue`

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
    <div class="content">
      <p>{{ messageText }}</p>
    </div>
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
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
    message: string;
    messageVars?: Record<string, unknown>;
    dismissable?: boolean;
  }>(),
  {
    titleVars: () => ({}),
    messageVars: () => ({}),
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
const messageText: string = te(props.message) ? t(props.message, props.messageVars) : props.message;

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

.content p {
  margin: 0;
  font-size: 14px;
}
</style>
```

### Updated Usage in `useSync.ts`

Update the usage in `useSync.ts` to pass a generic message:

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
        toastService.update(syncProcessingId.value!, {
          message: 'sync.toaster.progress',
          messageVars: {
            done: newStatus.synchronizationProgress?.syncedModels.length || 0,
            total: newStatus.synchronizationProgress?.allModels.length || 0
          }
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
          message: 'sync.toaster.progress',
          messageVars: {
            done: newStatus.synchronizationProgress?.syncedModels.length || 0,
            total: newStatus.synchronizationProgress?.allModels.length || 0
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
  });
}
```

### Explanation

1. **New Prop (`message` and `messageVars`):**
   - Added `message` and `messageVars

-----
To ensure the `SpinnerToast` component is more generic and only displays synchronization progress when it is related to sync, we can introduce a prop to conditionally render the sync-related content. Here’s how we can update the `SpinnerToast.vue` component:

### Updated `SpinnerToast.vue`

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
    <div class="content">
      <p v-if="isSync">{{ t('sync.toaster.requested', { syncedBy: titleVars.syncedBy }) }}</p>
      <p v-if="isSync">{{ t('sync.toaster.progress', { done: syncedModels.length, total: allModels.length }) }}</p>
      <slot v-else></slot>
    </div>
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
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
    isSync?: boolean;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    allModels: () => [],
    syncedModels: () => [],
    isSync: false
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

.content p {
  margin: 0;
  font-size: 14px;
}
</style>
```

### Explanation

1. **New Prop (`isSync`):**
   - A new prop `isSync` is added to determine if the toast is related to synchronization.
   - Default value for `isSync` is set to `false`.

2. **Conditional Rendering:**
   - The sync-related content (`t('sync.toaster.requested')` and `t('sync.toaster.progress')`) is only displayed if `isSync` is `true`.
   - A default slot is used for other content when `isSync` is `false`.

3. **Localization Keys:**
   - Ensure that your localization files include the necessary keys, as mentioned previously.

### Usage Example

When displaying a spinner toast related to sync, ensure `isSync` is set to `true`:

```typescript
toastService.display(ToastType.Spinner, {
  type: 'default',
  title: 'sync.toaster.requested',
  titleVars: {
    syncedBy: newStatus.synchronizationProgress?.syncedBy
  },
  allModels: newStatus.synchronizationProgress?.allModels || [],
  syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
  dismissable: true,
  isSync: true
});
```

For non-sync related spinners, simply omit the `isSync` prop or set it to `false`.

This approach ensures the `SpinnerToast` component remains generic and only shows sync-specific content when appropriate.





---------



You are correct that it's generally a good practice to ensure your branch is up-to-date with the main branch before creating a pull request. This helps avoid conflicts and ensures your changes are compatible with the latest codebase. Here’s a step-by-step process to update your branch and create a pull request:

1. **Switch to your main branch**:
   ```sh
   git checkout main
   ```

2. **Fetch the latest changes from the remote repository**:
   ```sh
   git fetch origin
   ```

3. **Pull the latest changes into your local main branch**:
   ```sh
   git pull origin main
   ```

4. **Switch back to your feature branch**:
   ```sh
   git checkout feature/117-new-feature
   ```

5. **Merge the latest changes from main into your feature branch**:
   ```sh
   git merge main
   ```

6. **Resolve any merge conflicts that arise**. If there are conflicts, Git will notify you, and you’ll need to manually resolve them in the affected files. After resolving conflicts, add the resolved files:
   ```sh
   git add <resolved_file>
   ```

7. **Complete the merge by committing the changes**:
   ```sh
   git commit
   ```

8. **Push the updated feature branch to the remote repository**:
   ```sh
   git push origin feature/117-new-feature
   ```

9. **Create a pull request** from your `feature/117-new-feature` branch to the `main` branch on your repository platform (e.g., GitHub, GitLab, Bitbucket).

This process ensures that your branch incorporates the latest updates from the main branch, reducing the likelihood of conflicts and making it easier for your pull request to be reviewed and merged.
----
### Explanation of Spread Operator

The spread operator (`...`) in JavaScript (and TypeScript) is used to create shallow copies of arrays or objects. In the context of your `displayWithId` method, `const properties = {...props}` creates a new object that contains all the properties of `props`. This is useful when you want to ensure that `properties` is a distinct object, even if it has the same properties as `props`.

#### Example:
```typescript
const props = { a: 1, b: 2 };
const properties = { ...props };

// properties is now { a: 1, b: 2 }, but it's a new object, not a reference to props
```

### Difference:
- **`const properties = {...props}`:** Creates a new object with the same properties as `props`. This ensures that any modifications to `properties` do not affect the original `props` object.
- **`const properties = props`:** Assigns the reference of `props` to `properties`. Any changes to `properties` will also affect `props` because they reference the same object.

In most cases, using the spread operator is a good practice to prevent unintended side effects by ensuring a new object is created. However, if you are certain that `props` won't be modified, you can simply assign `props` to `properties`.

### Updated `ToastService` with Comments

Here’s the updated `ToastService` with appropriate comments for the `displayWithId` and `update` methods, following the same format as other comments:

```typescript
import { DefineComponent } from 'vue';
import mitt, { Emitter, Handler } from 'mitt';
import { LoggingService } from '@ebitoolmx/logging-service/console';
import { loggingService as commonLoggingService } from './logging.js';

export enum ToastEvent {
  Display = 'display',
  Dismiss = 'dismiss',
  CloseAll = 'closeAll',
  Update = 'update' // Added the Update event
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
  [ToastEvent.Update]: { id: symbol; props: Record<string, unknown> }; // Added Update event type
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
   *
   * @param id - The unique identifier for the toast.
   * @param type - The type of toast to be displayed.
   * @param props - Additional properties to be passed to the toast component.
   * @param timeout - The duration (in milliseconds) for which the toast should be displayed.
   * @returns The unique identifier for the toast.
   */
  public displayWithId(
    id: symbol,
    type: ToastType,
    props: Record<string, unknown> = {},
    timeout: number | undefined = undefined
  ): symbol {
    const properties = { ...props }; // Spreading props to ensure a new object is created
    const component = this.toastComponents.get(type);

    if (!component) this.loggingService.error(`Toast type '${type}' has not been registered.`);

    this.events.emit(ToastEvent.Display, { id, component, properties, timeout });
    this.loggingService.debug(`Displaying toast of type '${type}'`, properties);

    return id;
  }

  /**
   * Displays a toast of a specified type. This will auto generate an id and return it before
   * emitting an event to the Toaster component. See displayWithId().
   *
   * @param type - The type of toast to be displayed.
   * @param props - Additional properties to be passed to the toast component.
   * @param timeout - The duration (in milliseconds) for which the toast should be displayed.
   * @returns The unique identifier for the toast.
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
   *
   * @param id - The unique identifier of the toast to be updated.
   * @param props - The new properties to be merged with the existing properties of the toast.
   */
  public update(id: symbol, props: Record<string, unknown>): void {
    this.events.emit(ToastEvent.Update, { id, props });
    this.loggingService.debug(`Updating toast with id '${id}'`, props);
  }

  /**
   * Dismisses a toast with the provided id if it is still present on the screen.
   *
   * @param id - The unique identifier of the toast to be dismissed.
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

### Summary

- **Spread Operator:** Using the spread operator ensures a new object is created, preventing unintended side effects.
- **Comments:** Added comments to the `displayWithId` and `update` methods following the existing format for consistency.

By following these updates, your `ToastService` will handle updates correctly, and the comments will provide clear documentation for future maintenance and understanding.


------
To combine both messages into one and ensure the progress message is shown, we need to adjust how the message is composed and displayed in `Spinner.vue`. We will concatenate the existing message (`titleText`) and the progress message (`progressMessage`).

Here is the updated code for `Spinner.vue`:

1. Combine `titleText` and `progressMessage` into a single computed property.
2. Display this combined message in the template.

### Updated `Spinner.vue`

```vue
<template>
  <div class="item default">
    <div class="header">
      <span class="title semi-bold">{{ combinedMessage }}</span>
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
import { ref, onBeforeUnmount, onMounted, computed } from 'vue';

import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';

import { ToastEvent, toastService } from '@/services/toast.js';
import { useI18n } from 'vue-i18n';
import { useSyncStore } from '@/stores/syncStore'; // Import the sync store

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

const syncStore = useSyncStore(); // Use the sync store

const progressMessage = computed(() => {
  if (syncStore.status) {
    return `${syncStore.status.syncedModels.length}/${syncStore.status.allModels.length} models are done.`;
  }
  return '';
});

const combinedMessage = computed(() => {
  return `${titleText}\n${progressMessage.value}`;
});

onMounted(() => {
  console.log('Spinner mounted and listening for events');
  toastService.on(ToastEvent.Dismiss, onDismiss);
});

onBeforeUnmount(() => {
  console.log('Spinner before unmount and removing event listeners');
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
```

### Explanation

1. **Combined Message:**
   - The `combinedMessage` computed property concatenates `titleText` and `progressMessage`. This ensures both messages are displayed together in the toast.

2. **Template Update:**
   - The template now uses `combinedMessage` instead of `titleText`.

This approach ensures both the original title message and the sync progress message are displayed in the same toast body. The `combinedMessage` property dynamically updates whenever `titleText` or `progressMessage` changes, ensuring real-time updates in the toast.


----------
Let's debug and ensure that the `Sync.vue` component properly updates the Pinia store. Here are the steps to ensure correct interaction between `Sync.vue` and the `syncProgressStore`:

1. **Ensure Pinia Store Setup in `Sync.vue`**: Ensure the `syncProgressStore` is correctly imported and used in `Sync.vue`.

2. **Correctly Update the Store in `Sync.vue`**: Make sure the store's state is updated correctly during the sync process.

3. **Verify Store State Updates in `Spinner.vue`**: Confirm that `Spinner.vue` properly reflects updates from the store.

### Step-by-Step Implementation:

#### 1. Ensure Pinia Store Setup in `Sync.vue`

Ensure the Pinia store is correctly imported and used in `Sync.vue`.

**Sync.vue**:
```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { useI18n } from 'vue-i18n';
import { useSyncProgressStore } from '@/stores/syncProgress'; // Ensure correct path
import { useAuthService } from '@/auth/index.js';
import { useAppStore } from '@/stores/app.js';
import { useProductsStore } from '@/stores/products.js';
import { useConnectionsStore } from '@/stores/connections.js';
import { useEditorStore } from '@/stores/editor.js';
import { toastService, ToastType } from '@/services/toast.js';
import { SyncDialogType } from '@/typings/sync.js';
import { DialogNames } from '@/typings/dialog.js';
import { eventService, EventType } from '@/services/event.js';
import AppIcon from '@/components/common/icon/Icon.vue';
import AppButton from '@/components/common/formElements/button/Button.vue';
import AppDialogContainer from '@/components/common/dialog/layouts/DialogContainer.vue';
import AppDialogIconContentLayout, { IconColor } from '@/components/common/dialog/layouts/DialogIconContentLayout.vue';
import AppDialogProcessContentLayout from '@/components/common/dialog/layouts/DialogProcessContentLayout.vue';
import { LocaleMessage } from '@/locale/en.js';
import { MXSyncProgressEvent, MXSyncProgressStatus } from '@ebitoolmx/gateway-types';

const syncProgressStore = useSyncProgressStore();
const { userDetails } = useAuthService();
const appStore = useAppStore();
const connectionsStore = useConnectionsStore();
const productsStore = useProductsStore();
const editorStore = useEditorStore();
const { t } = useI18n();

const props = defineProps<{
  dialogType: SyncDialogType;
  status?: MXSyncProgressEvent;
}>();

const emit = defineEmits(['close']);

const header = computed(() => {
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
      return '';
  }
});

const message = computed(() => {
  switch (props.dialogType) {
    case SyncDialogType.Requested:
      return 'sync.modal.message.requested';
    case SyncDialogType.Processing:
      return 'sync.modal.message.progress';
    case SyncDialogType.Offline:
      return 'sync.modal.message.offline';
    case SyncDialogType.Initialising:
    case SyncDialogType.Successful:
    case SyncDialogType.InitialisingFailed:
    case SyncDialogType.Unsuccessful:
    default:
      return null;
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
    case SyncDialogType.InitialisingFailed:
      return 'danger';
    case SyncDialogType.Offline:
      return 'warning';
    default:
      return 'text';
  }
});

const isSyncRequestDialog = computed<boolean>(() => props.dialogType === SyncDialogType.Requested);
const isSyncProcessingDialog = computed<boolean>(() => props.dialogType === SyncDialogType.Processing || props.dialogType === SyncDialogType.Initialising);
const isSyncSuccessfulDialog = computed<boolean>(() => props.dialogType === SyncDialogType.Successful);
const isSyncUnsuccessfulDialog = computed<boolean>(() => props.dialogType === SyncDialogType.Unsuccessful);
const isSyncFailedDialog = computed<boolean>(() => props.dialogType === SyncDialogType.InitialisingFailed);
const startDisabled = computed<boolean>(() => appStore.hasBlockingTasks || props.status?.syncProgressStatus === MXSyncProgressStatus.Synchronizing);

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Update shared state with initial progress data
  const totalModels = 5;
  const syncedModels = 5;

  syncProgressStore.setTotalModels(totalModels);
  syncProgressStore.setSyncedModels(syncedModels);
}

watch(() => props.status, (newStatus) => {
  if (newStatus) {
    syncProgressStore.setTotalModels(newStatus.allModels.length);
    syncProgressStore.setSyncedModels(newStatus.syncedModels.length);
    if (userDetails.value?.email !== newStatus.initiator) {
      toastService.display(ToastType.Spinner, {
        title: 'Synchronising...',
        syncedModels: newStatus.syncedModels.length,
        totalModels: newStatus.allModels.length,
      });
    }
  }
}, { immediate: true, deep: true });

watch(() => connectionsStore.isOffline, (nowOffline) => {
  if (nowOffline) {
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
});
</script>

<template>
  <app-dialog-container title-key="sync.modal.title" data-testid="dialog-sync">
    <template #content>
      <app-dialog-process-content-layout v-if="isSyncProcessingDialog" :header="t(header)" :description-line1="message ? t(message) : message">
        <template #extra-content>
          <ul class="sync-progress lead dialog-content-message">
            <li v-for="(model, index) of syncProgressStore.total" :key="index" :class="{ syncing: syncProgressStore.synced.includes(model), synced: syncProgressStore.synced.includes(model) }">
              <app-icon :name="progressIcon(model)" />
              {{ model }}
            </li>
          </ul>
        </template>
      </app-dialog-process-content-layout>

      <app-dialog-icon-content-layout v-else :icon="icon" :icon-color="iconColor" :header="t(header)" :message-line1="message ? t(message) : message">
        <template #extra-content>
          <div v-if="isSyncUnsuccessfulDialog || isSyncFailedDialog" class="unsuccessful">
            <div class="unsuccessful-message">
              <p v-if="isSyncUnsuccessfulDialog" class="lead dialog-content-message">{{ t('sync.modal.message.unsuccessful') }}</p>
              <p class="lead dialog-content-message">{{ t('sync.modal.message.contact') }}</p>
              <a :href="t('common.link')" rel="noopener noreferrer" target="_blank" class="lead dialog-content-message">{{ t('common.helpSupport') }}</a>
              <p class="lead dialog-content-message">.</p>
            </div>

            <div v-if="syncProgressStore.currentModelName && isSyncUnsuccessfulDialog" class="failed-model">
              <app-icon name="sync_error" />
              <p class="lead dialog-content-message">{{ syncProgressStore.currentModelName }}</p>
            </div>
          </div>
          <div v-else-if="isSyncSuccessfulDialog">
            <ul class="sync-progress lead dialog-content-message">
              <li v-for="(model, index) of syncProgressStore.synced" :key="index" class="synced">
                <app-icon name="sync_success" />
                {{ model }}
              </li>
            </ul>
          </div>
        </template>
      </app-dialog-icon-content-layout>
    </template>

    <template #button-row>
      <app-button data-testid="reject-sync" :disabled="appStore.hasBlockingTasks" :class="{ primary: !isSyncRequestDialog, secondary: isSyncRequestDialog }" @click="emit('close')">{{ t('common.close') }}</app-button>
      <app

-button v-if="isSyncRequestDialog" data-testid="accept-sync" class="primary" :disabled="startDisabled" @click="start">{{ t('common.start') }}</app-button>
    </template>
  </app-dialog-container>
</template>

<style scoped>
.sync-progress {
  color: var(--font-disabled-color);
  fill: var(--font-disabled-color);

  li {
    display: flex;
    align-items: center;

    .icon {
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
    fill: var(--success);
  }
}

.unsuccessful {
  display: flex;
  flex-direction: column;
  align-items: center;

  .unsuccessful-message {
    display: flex;

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }
}
</style>
```

### Summary:
1. **Sync.vue**: Ensures the Pinia store is correctly imported and updates the store during the sync process.
2. **SyncProgressStore**: Properly stores and manages sync progress.
3. **Spinner.vue**: Reflects updates from the Pinia store.

Please test these changes and let me know if there are any further issues or if adjustments are needed.
--------------

Given that your project uses Pinia for state management, let's refactor the `syncProgressState` into a Pinia store to align with your project's structure.

### Step-by-Step Implementation

1. **Create a Pinia Store for Sync Progress**
2. **Update `Sync.vue` to Use the Pinia Store**
3. **Update `Spinner.vue` to Use the Pinia Store**

### Step 1: Create a Pinia Store for Sync Progress

Create a new file `syncProgressStore.ts` in the `stores` directory.

**syncProgressStore.ts:**
```ts
import { defineStore } from 'pinia';
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
    setSyncedModels,
  };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useSyncProgressStore, import.meta.hot));
}
```

### Step 2: Update `Sync.vue` to Use the Pinia Store

In `Sync.vue`, import and use the Pinia store to update the synchronization progress.

**Sync.vue:**
```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { PickupPaths, useI18n } from 'vue-i18n';
import { useSyncProgressStore } from '@/stores/syncProgressStore'; // Import the Pinia store

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
const editorStore = useEditorStore();
const { t } = useI18n();

const syncProgressStore = useSyncProgressStore(); // Use the Pinia store

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Update shared state with initial progress data
  syncProgressStore.setTotalModels(props.status?.allModels.length || 0);
  syncProgressStore.setSyncedModels(props.status?.syncedModels.length || 0);
}

watch(
  () => props.status,
  (newStatus) => {
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
```

### Step 3: Update `Spinner.vue` to Use the Pinia Store

In `Spinner.vue`, import and use the Pinia store to get the synchronization progress.

**Spinner.vue:**
```vue
<script setup lang="ts">
import { computed } from 'vue';
import { useSyncProgressStore } from '@/stores/syncProgressStore'; // Import the Pinia store

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

const syncProgressStore = useSyncProgressStore(); // Use the Pinia store

const dismiss = () => emit('dismiss', props.id);

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;
const progressText = computed(() => `${syncProgressStore.synced} / ${syncProgressStore.total} models are done`); // Computed property for progress text

onMounted(() => {
  toastService.on(ToastEvent.Dismiss, onDismiss);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.Dismiss, onDismiss);
});
</script>
```

### Summary

1. **Created a Pinia store for synchronization progress (`syncProgressStore.ts`).**
2. **Updated `Sync.vue` to use the Pinia store for updating synchronization progress.**
3. **Updated `Spinner.vue` to use the Pinia store for displaying synchronization progress.**

This setup aligns with your project's state management approach using Pinia and ensures that the synchronization progress is correctly updated and displayed. If you encounter any issues or need further adjustments, please let me know!


============================


#Understood. Let's focus on ensuring the shared state (`syncProgressState`) is updated correctly in `Sync.vue`. We will verify the part where we update the state directly in the `start` function and the watcher.

### Step-by-Step Fix

1. **Ensure the Shared State is Imported Correctly**: Verify the import statement.
2. **Update the Shared State in the `start` Function**: Verify the state updates.
3. **Use the Watcher to Update the Shared State**: Verify reactivity.

### Ensure the Shared State is Imported Correctly

First, make sure the import statement is correct and placed at the top of `Sync.vue`:

```ts
import { syncProgressState } from '@/store/syncProgressState';
```

### Update the Shared State in the `start` Function

Ensure the state updates in the `start` function are correctly updating the reactive state. Here’s the updated part:

```ts
function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Update shared state with initial progress data
  syncProgressState.totalModels = props.status?.allModels.length || 0;
  syncProgressState.syncedModels = props.status?.syncedModels.length || 0;
}
```

Place this function definition within your `<script setup>` block:

**Sync.vue:**
```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { PickupPaths, useI18n } from 'vue-i18n';
import { syncProgressState } from '@/store/syncProgressState'; // Import the shared state

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
const editorStore = useEditorStore();
const { t } = useI18n();

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Update shared state with initial progress data
  syncProgressState.totalModels = props.status?.allModels.length || 0;
  syncProgressState.syncedModels = props.status?.syncedModels.length || 0;
}
```

### Use the Watcher to Update the Shared State

Ensure the watcher updates the state reactively:

**Sync.vue:**
```ts
watch(
  () => props.status,
  (newStatus) => {
    if (newStatus) {
      // Update shared state whenever the status changes
      syncProgressState.totalModels = newStatus.allModels.length;
      syncProgressState.syncedModels = newStatus.syncedModels.length;
    }
  },
  { immediate: true, deep: true }
);
```

Place this watcher definition within your `<script setup>` block:

**Sync.vue:**
```vue
<script setup lang="ts">
import { watch, computed } from 'vue';
import { PickupPaths, useI18n } from 'vue-i18n';
import { syncProgressState } from '@/store/syncProgressState'; // Import the shared state

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
const editorStore = useEditorStore();
const { t } = useI18n();

function start() {
  const email = userDetails.value?.email;
  if (!email) return;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Update shared state with initial progress data
  syncProgressState.totalModels = props.status?.allModels.length || 0;
  syncProgressState.syncedModels = props.status?.syncedModels.length || 0;
}

watch(
  () => props.status,
  (newStatus) => {
    if (newStatus) {
      // Update shared state whenever the status changes
      syncProgressState.totalModels = newStatus.allModels.length;
      syncProgressState.syncedModels = newStatus.syncedModels.length;
    }
  },
  { immediate: true, deep: true }
);
</script>
```

### Verification

- **Ensure the `start` function is triggered correctly.**
- **Ensure the watcher updates the `syncProgressState` whenever `props.status` changes.**

If the shared state is still not updating, try logging the values to debug:

```ts
watch(
  () => props.status,
  (newStatus) => {
    if (newStatus) {
      console.log('New Status:', newStatus);
      console.log('Updating syncProgressState');
      syncProgressState.totalModels = newStatus.allModels.length;
      syncProgressState.syncedModels = newStatus.syncedModels.length;
    }
  },
  { immediate: true, deep: true }
);
```

This should help ensure the state is updated correctly and reflect changes in `Spinner.vue`. If you encounter further issues, please let me know!

=============

Let's address the issue with updating the shared state (`syncProgressState`) from `Sync.vue`. It seems that the values are not being updated properly. We will ensure that the reactive state is correctly updated from `Sync.vue`.

### Steps to Ensure Proper State Update

1. **Verify the Imports and Paths**: Ensure that the import paths are correct in both `Sync.vue` and `Spinner.vue`.
2. **Ensure Reactivity in the Shared State**: Use `reactive` to make sure the shared state is reactive and can be updated.
3. **Update the Shared State Properly in `Sync.vue`**: Use appropriate Vue reactivity methods to update the state.

### Verify Imports and Paths

Ensure that the paths to `syncProgressState` are correct:

**Sync.vue:**
```ts
import { syncProgressState } from '@/store/syncProgressState';
```

**Spinner.vue:**
```ts
import { syncProgressState } from '@/store/syncProgressState';
```

### Ensure Reactivity in the Shared State

**syncProgressState.ts:**
```ts
import { reactive } from 'vue';

export const syncProgressState = reactive({
  totalModels: 0,
  syncedModels: 0,
});
```

### Update the Shared State Properly in `Sync.vue`

In `Sync.vue`, ensure that we are correctly updating the shared state when synchronization status changes.

**Sync.vue:**
```vue
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

import { syncProgressState } from '@/store/syncProgressState'; // Import the shared state

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
  syncProgressState.totalModels = props.status?.allModels.length || 0;
  syncProgressState.syncedModels = props.status?.syncedModels.length || 0;
}

watch(
  () => props.status,
  (newStatus) => {
    if (newStatus) {
      // Update shared state whenever the status changes
      syncProgressState.totalModels = newStatus.allModels.length;
      syncProgressState.syncedModels = newStatus.syncedModels.length;
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
```

### Verify Reactivity in the Shared State

Ensure `syncProgressState` is correctly set up for reactivity:

**syncProgressState.ts:**
```ts
import { reactive } from 'vue';

export const syncProgressState = reactive({
  totalModels: 0,
  syncedModels: 0,
});
```

### Step 3: Modify `Spinner.vue` to Display Sync Progress from Shared State

Ensure `Spinner.vue` correctly displays values from `syncProgressState`:

**Spinner.vue:**
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
    <p class="progress-text">{{ progressText }}</p> <!-- New line for progress text -->
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, computed } from 'vue';
import { syncProgressState } from '@/store/syncProgressState'; // Import the shared state

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

const dismiss = () => emit('dismiss', props.id);

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;
const progressText = computed(() => `${syncProgressState.syncedModels} / ${syncProgressState.totalModels} models are done`); // Computed property for progress text

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
  padding-right: var (--base-spacing-0);
}

.header .dismiss {
  fill: var (--toaster-dismiss-icon);
  flex-shrink: 0;
  height: 16px;
  width: 16px;
  font-size: 16px;
  display: inline-block;
}

.header .dismiss:hover {
  fill: var (--toaster-dismiss-icon);
}

.spinner {
  margin: 15px 0 15px;
}

.progress-text {  // Style for progress text
  margin-top: var (--base-spacing-2);
  font-size: var (--font-size-small);
}
</style>
```

### Summary

- **Ensure `syncProgressState` is correctly imported and updated in `Sync.vue`.**
- **Verify the state updates properly in `Sync.vue` using watchers.**
- **Ensure `Spinner.vue` correctly displays the progress values from the shared state.**

This setup should ensure the synchronization progress is updated and displayed correctly for collaborators. If you encounter any issues or need further adjustments, please let me know!





-----------

You should place the `syncProgressState.ts` file in a logical location within your project's directory structure. Typically, it would be placed in a directory that holds shared state or store files. Here is an example structure:

```
src/
├── components/
│   ├── common/
│   │   ├── icon/
│   │   ├── spinner/
│   │   ├── IconButton.vue
│   │   └── HorizontalSpinner.vue
│   ├── toast/
│   │   └── Spinner.vue
├── services/
│   ├── toast.js
│   └── logging.js
├── store/                     # A logical place for shared state
│   └── syncProgressState.ts   # Shared state file
├── views/
│   ├── Sync.vue
├── main.js
```

In this structure, the `store` directory is used for managing shared state across components, and the `syncProgressState.ts` file would be placed inside it.

### Creating `syncProgressState.ts`

Create a new file named `syncProgressState.ts` inside the `store` directory with the following content:

**syncProgressState.ts:**
```ts
import { reactive } from 'vue';

export const syncProgressState = reactive({
  totalModels: 0,
  syncedModels: 0,
});
```

### Updating Imports

Now, make sure you update the import paths in your components to reflect the correct location of `syncProgressState.ts`.

**Sync.vue:**
```ts
import { syncProgressState } from '@/store/syncProgressState'; // Update the import path
```

**Spinner.vue:**
```ts
import { syncProgressState } from '@/store/syncProgressState'; // Update the import path
```

### Example of the File Structure

Here’s how the file structure would look with the new shared state file:

```
src/
├── components/
│   ├── common/
│   │   ├── icon/
│   │   │   └── IconButton.vue
│   │   ├── spinner/
│   │   │   └── HorizontalSpinner.vue
│   │   └── spinner/
│   │       └── Spinner.vue
├── services/
│   ├── toast.js
│   └── logging.js
├── store/
│   └── syncProgressState.ts   # Shared state file
├── views/
│   ├── Sync.vue
├── main.js
```

By following this structure, you ensure that your shared state is organized and easily accessible across your application components. If you have any further questions or need additional help, feel free to ask!






================================================
Got it. Let's focus on passing the synchronization data (`totalModels` and `syncedModels`) to the `Spinner.vue` component directly without using any toast notifications in `Sync.vue`.

### Step-by-Step Implementation

1. **Create a shared state to store the synchronization progress.**
2. **Update `Sync.vue` to update the shared state with synchronization progress.**
3. **Modify `Spinner.vue` to display synchronization progress from the shared state.**

### Step 1: Create a Shared State for Synchronization Progress

Create a new file `syncProgressState.ts` to manage the synchronization state.

**syncProgressState.ts:**
```ts
import { reactive } from 'vue';

export const syncProgressState = reactive({
  totalModels: 0,
  syncedModels: 0,
});
```

### Step 2: Update `Sync.vue` to Update Shared State

**Sync.vue:**
```vue
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

import { syncProgressState } from '@/path/to/syncProgressState'; // Import the shared state

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
  syncProgressState.totalModels = props.status?.allModels.length || 0;
  syncProgressState.syncedModels = props.status?.syncedModels.length || 0;
}

watch(
  () => props.status,
  (newStatus) => {
    if (newStatus) {
      // Update shared state whenever the status changes
      syncProgressState.totalModels = newStatus.allModels.length;
      syncProgressState.syncedModels = newStatus.syncedModels.length;
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
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
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
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }
}
</style>
```

### Step 3: Modify `Spinner.vue` to Display Sync Progress from Shared State

**Spinner.vue:**
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
    <p class="progress-text">{{ progressText }}</p> <!-- New line for progress text -->
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, computed } from 'vue';
import { syncProgressState } from '@/path/to/syncProgressState'; // Import the shared state

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

const dismiss = () => emit('dismiss', props.id);

const onDismiss = (id: symbol) => {
  if (id === props.id) {
    dismiss();
  }
};

const titleText: string = te(props.title) ? t(props.title, props.titleVars) : props.title;
const progressText = computed(() => `${syncProgressState.syncedModels} / ${syncProgressState.totalModels} models are done`); // Computed property for progress text

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

.progress-text {  // Style for progress text
  margin-top: var(--base-spacing-2);
  font-size: var(--font-size-small);
}
</style>
```

### Summary

1. **Created a shared state `syncProgressState` to store the synchronization progress.**
2. **Updated `Sync.vue` to update the shared state with synchronization progress data.**
3. **Modified `Spinner.vue` to display the synchronization progress using the shared state.**

This way, the synchronization progress will be displayed in the `Spinner.vue` component for collaborators without triggering any toast notifications from `Sync.vue`. If you need further adjustments or clarification, please let me know!










#
Sure, here’s a detailed overview of the Halo universe lore up to the start of Master Chief’s story:

Ancient History

The Precursors and the Forerunners

	•	Precursors: The Precursors were an ancient and advanced race believed to have created the Milky Way galaxy and its various species, including humans and Forerunners. They held the Mantle of Responsibility, a philosophy of stewardship over all life.
	•	Forerunners: The Forerunners were another ancient and highly advanced species. They revered the Precursors and inherited the Mantle of Responsibility, aiming to protect and cultivate life in the galaxy.

The Flood

	•	The Flood: A parasitic species capable of infecting and controlling other life forms. The Precursors, after being nearly exterminated by the Forerunners, transformed into the Flood as an act of vengeance.

The Forerunner-Flood War

	•	War: The Flood emerged as a major threat to the galaxy, leading to a prolonged and devastating war with the Forerunners. The Flood’s ability to infect and control sentient beings made them a nearly unstoppable force.

Halo Array

	•	Creation: In a desperate bid to stop the Flood, the Forerunners created the Halo Array, a series of massive ring-shaped superweapons. These rings were designed to wipe out all sentient life in the galaxy, starving the Flood of hosts.
	•	Activation: The Forerunners eventually activated the Halo Array, sacrificing themselves and all sentient life in the galaxy to stop the Flood. This event is known as the Great Purification.

Human Prehistory and Rise

Ancient Humanity

	•	Human-Forerunner War: Long before the Halo Array was activated, ancient humans were a space-faring species. They came into conflict with the Forerunners due to misunderstandings and the threat posed by the Flood, leading to a devastating war.
	•	Downfall: The Forerunners ultimately defeated humanity, regressing them to a primitive state and scattering them across the galaxy.

Reclamation

	•	Librarian and Didact: Two prominent Forerunners, the Librarian and the Didact, played key roles in preserving species and ensuring that humanity would eventually rise again. The Librarian seeded human worlds with genetic imprints to guide their evolution.

The Covenant

Formation

	•	Covenant: An alliance of alien species united by the worship of the Forerunners and their artifacts. The Covenant is led by the Prophets and includes species such as the Elites, Brutes, Grunts, Jackals, and others.
	•	Religious Zeal: The Covenant believes that activating Forerunner technology will lead them on a Great Journey, a path to transcendence and divinity.

Expansion and War

	•	Expansion: The Covenant rapidly expands across the galaxy, discovering and activating Forerunner artifacts. They encounter humanity, whom they view as heretics to be exterminated.
	•	Human-Covenant War: This leads to a brutal conflict known as the Human-Covenant War, with humanity struggling to survive against the technologically superior Covenant forces.

Rise of the UNSC and SPARTAN Program

United Nations Space Command (UNSC)

	•	Formation: The UNSC is the military, exploratory, and scientific agency of humanity, tasked with defending human colonies and exploring space.
	•	Colonial Conflicts: Prior to the Covenant War, humanity faced internal strife with colonies seeking independence from Earth, leading to a series of insurrections.

SPARTAN Program

	•	SPARTAN-II: In response to both the insurrectionists and the emerging Covenant threat, Dr. Catherine Halsey initiated the SPARTAN-II program. This involved kidnapping and training children to become super-soldiers through rigorous training and augmentation.
	•	SPARTAN-IIs: These soldiers, including John-117 (Master Chief), are physically and mentally enhanced to be the ultimate warriors, capable of taking on overwhelming odds.

Halo Timeline Up to Master Chief

The Covenant Attack

	•	First Contact: Humanity’s first contact with the Covenant occurs in 2525, leading to the destruction of the colony on Harvest. This marks the beginning of the Human-Covenant War.
	•	Escalation: The Covenant systematically attacks and destroys human colonies, pushing humanity to the brink of extinction.

Master Chief’s Early Years

	•	Training: John-117, known as Master Chief, emerges as one of the most capable and effective SPARTAN-IIs. His early missions establish his reputation as a skilled and resourceful soldier.
	•	Reach: The fall of Reach, a major human stronghold, is a critical moment in the war. Despite the planet’s fall, Master Chief and other survivors manage to escape, setting the stage for the events of the first Halo game.

Prelude to Halo: Combat Evolved

	•	Pillar of Autumn: As the war continues, Master Chief and the crew of the UNSC ship Pillar of Autumn discover a mysterious ring world, later known as Halo. This discovery marks the beginning of the events in “Halo: Combat Evolved,” where Master Chief’s legend truly begins.

This timeline provides a detailed look at the Halo universe leading up to the start of Master Chief’s story, highlighting key events and developments that shape the series’ rich lore.




Comparing a SPARTAN from the Halo universe to a Space Marine from the Warhammer 40k universe is an intriguing matchup, given the substantial differences in their respective settings, enhancements, and equipment. Here’s a breakdown of their capabilities and how they might fare in a one-on-one fight:

### SPARTAN-II (Master Chief or Noble Six)
- **Enhancements:** SPARTAN-IIs undergo extensive physical and mental enhancements from a young age, making them stronger, faster, and more resilient than normal humans. They have enhanced reflexes, strength, agility, and healing capabilities.
- **Armor:** They wear Mjolnir Powered Assault Armor, which provides them with augmented strength, energy shields, advanced targeting systems, and integrated AI (like Cortana for Master Chief). The armor also has environmental protection and various vision modes.
- **Weapons:** SPARTANs are proficient with a wide array of weapons, including the MA5 series assault rifles, M6 magnums, and various Covenant and Forerunner technology. They also use advanced hand-to-hand combat techniques.

### Space Marine (Warhammer 40k)
- **Enhancements:** Space Marines undergo genetic modification and extensive training to become superhuman warriors. They are implanted with various organs that enhance their strength, speed, endurance, and healing abilities. They are significantly larger and stronger than regular humans.
- **Armor:** Space Marines wear Power Armor, which includes features like enhanced strength, a life support system, and various sensors. Their armor is incredibly durable and can withstand extreme damage.
- **Weapons:** Space Marines wield powerful weapons such as Bolters, which fire explosive rounds, Chainswords for close combat, and heavy weapons like plasma guns and melta weapons. They also have a wealth of wargear and equipment designed for various combat scenarios.

### Combat Scenario Analysis

#### Physical Attributes
- **Strength and Durability:** Space Marines generally have the edge in raw physical strength and durability due to their larger size and extensive genetic modifications. Their Power Armor is also heavily fortified.
- **Agility and Reflexes:** SPARTANs might have an advantage in terms of agility and reflexes, given their training and the advanced capabilities of the Mjolnir armor.

#### Equipment and Technology
- **Weapons:** Space Marine weapons are typically more powerful and designed for high-impact damage, capable of devastating effects. However, SPARTANs have access to a wide range of advanced and versatile weaponry, including Covenant and Forerunner tech.
- **Armor:** Both have incredibly advanced armor systems. Space Marine Power Armor offers substantial protection and strength augmentation, while Mjolnir armor provides enhanced agility, shielding, and tactical advantages with AI integration.

#### Tactics and Training
- **Training:** Both SPARTANs and Space Marines are elite soldiers with rigorous training and combat experience. SPARTANs are trained from childhood to be the perfect soldiers, while Space Marines undergo intense training and indoctrination to become superhuman warriors.
- **Adaptability:** SPARTANs are known for their tactical flexibility and ability to adapt to various combat situations. Space Marines are similarly versatile but often operate with a more rigid strategic doctrine.

### Outcome
While it's difficult to definitively declare a winner given the differences in their respective universes, here are a few possible scenarios:

1. **Close Combat:** In a pure melee fight, the Space Marine's superior strength and the brutal effectiveness of weapons like the Chainsword could give them an edge over a SPARTAN. However, a SPARTAN's agility and combat training might allow them to outmaneuver and exploit weak points.

2. **Ranged Combat:** In a ranged battle, the outcome could vary depending on the specific weapons used. Space Marine Bolters are devastating, but SPARTANs could leverage their energy shields, advanced targeting systems, and possibly superior tactical planning to gain an advantage.

3. **Environmental Factors:** The environment and terrain could play a significant role. SPARTANs, with their agility and ability to quickly adapt, might perform better in varied and complex environments, while Space Marines would dominate in more straightforward, brutal confrontations.

Ultimately, the battle could go either way depending on numerous variables, but in general:
- **Space Marine:** Has the edge in raw power, durability, and heavy weaponry.
- **SPARTAN:** Has the edge in agility, reflexes, advanced technology, and tactical flexibility.

The match-up is highly situational, and either combatant could potentially claim victory under the right circumstances.


Master Chief (John-117) and Noble Six (SPARTAN-B312) are among the most prominent and capable SPARTANs in the Halo universe. Here’s an in-depth look at their strengths and the capabilities of other SPARTANs who are on a similar level.

### Master Chief (John-117)

#### Physical and Mental Enhancements
- **Strength:** Master Chief's augmentations give him superhuman strength, allowing him to lift several times his body weight. This strength is further amplified by his Mjolnir armor.
- **Speed and Agility:** He can run at speeds up to 60 km/h (37 mph) and has reaction times measured in milliseconds.
- **Resilience:** Enhanced bones make him nearly unbreakable, and his enhanced healing allows for rapid recovery from injuries.
- **Mental Acuity:** His brain functions at a highly accelerated rate, making him an exceptional tactician and combatant.

#### Armor and Equipment
- **Mjolnir Armor:** The Mark VI Mjolnir armor boosts his strength, speed, and reflexes, and provides energy shielding capable of withstanding significant damage.
- **Weapons:** Proficient with a wide range of human and Covenant weaponry, from assault rifles to energy swords.

### Noble Six (SPARTAN-B312)

#### Physical and Mental Enhancements
- **Comparable to Master Chief:** Noble Six possesses similar physical enhancements, making them extremely strong, fast, and resilient.
- **Combat Skills:** Known for exceptional combat prowess and the ability to adapt quickly to changing battle conditions. Six's abilities are on par with, if not sometimes exceeding, other SPARTAN-IIIs due to their extensive experience and skill.

#### Armor and Equipment
- **Mjolnir Armor:** Equipped with the latest versions of Mjolnir armor during the Fall of Reach, including energy shielding and advanced tactical systems.
- **Weapons:** Skilled with a variety of weapons, from conventional firearms to alien technology.

### Other Notable SPARTANs

#### SPARTAN-IIs
1. **Kelly-087**
   - **Strengths:** Known for her incredible speed and agility, often considered the fastest of the SPARTAN-IIs.
   - **Role:** Recon and close-quarters combat.

2. **Fred-104**
   - **Strengths:** Highly skilled in leadership and tactics, second only to Master Chief in combat ability.
   - **Role:** Often serves as the second-in-command in Blue Team.

3. **Linda-058**
   - **Strengths:** Exceptional marksmanship and stealth capabilities, known as the best sniper among the SPARTAN-IIs.
   - **Role:** Sniper and reconnaissance.

#### SPARTAN-IIIs
1. **Emile-A239**
   - **Strengths:** Expert in close-quarters combat, known for his ruthlessness and skill with a kukri blade.
   - **Role:** Assault and breaching operations.

2. **Jun-A266**
   - **Strengths:** Skilled sniper and scout, capable of operating independently for extended periods.
   - **Role:** Reconnaissance and long-range engagement.

3. **Carter-A259**
   - **Strengths:** Strong leadership skills and tactical planning, commanding Noble Team during the Fall of Reach.
   - **Role:** Team leader and strategist.

### Comparison and Summary

- **Physical Enhancements:** All SPARTANs undergo rigorous physical enhancements that give them superhuman abilities. SPARTAN-IIs, including Master Chief and Kelly-087, are typically stronger and more resilient due to the more intensive augmentation process.
- **Armor and Equipment:** The Mjolnir armor significantly amplifies their physical capabilities, with each new generation of armor providing better shielding, mobility, and integrated systems.
- **Skills and Experience:** SPARTAN-IIs like Master Chief, Kelly, Fred, and Linda are known for their extensive combat experience and exceptional skills. SPARTAN-IIIs like Noble Six, Emile, and Jun are also highly skilled but were often used in high-risk operations with lower survival rates.

Master Chief and Noble Six stand out due to their exceptional combat records and leadership qualities. Master Chief's legendary status is built on decades of successful missions and his role in pivotal battles against the Covenant and the Flood. Noble Six's heroism during the Fall of Reach demonstrates their combat prowess and determination.

In conclusion, while Master Chief and Noble Six are among the most formidable SPARTANs, they are part of a larger cadre of elite soldiers, each with unique strengths that contribute to their effectiveness in various combat scenarios.

During the Human-Covenant War, the number of SPARTANs varied across different generations, and newer generations of SPARTANs and advanced armor systems have been developed over time. Here’s a detailed breakdown:

### SPARTAN Generations

#### SPARTAN-I Program
- **Overview:** The initial SPARTAN-I program, also known as the ORION Project, was an early attempt to create super-soldiers. The program had limited success and produced a small number of augmented soldiers.
- **Numbers:** Approximately 300 candidates, with fewer than 50 successful augmentations.

#### SPARTAN-II Program
- **Overview:** The SPARTAN-II program, initiated by Dr. Halsey, was more successful and created highly effective super-soldiers through extensive genetic and cybernetic augmentations.
- **Numbers:** Initially, 75 children were abducted and trained, but not all survived the augmentation process. Approximately 33 SPARTAN-IIs were fully operational at the start of the Covenant War. Over time, some were lost in battle, reducing their numbers further.

#### SPARTAN-III Program
- **Overview:** The SPARTAN-III program was designed to produce more SPARTANs at a lower cost, using less intensive augmentations. These soldiers were often deployed in high-risk missions with high casualty rates.
- **Numbers:** Three main companies were created:
  - **Alpha Company:** Approximately 300 SPARTAN-IIIs.
  - **Beta Company:** Approximately 300 SPARTAN-IIIs.
  - **Gamma Company:** Approximately 330 SPARTAN-IIIs.
  - Not all SPARTAN-IIIs survived their missions, and many were considered expendable.

#### SPARTAN-IV Program
- **Overview:** The SPARTAN-IV program, initiated post-war, recruited adult volunteers from existing military personnel and provided them with advanced augmentations and armor.
- **Numbers:** The exact number is classified, but there are several hundred SPARTAN-IVs. This program aims to produce a larger number of SPARTANs with less intensive but still significant enhancements.

### Latest Armors and Weapons

#### Armor Systems

1. **Mjolnir GEN2 Armor**
   - **Features:** Enhanced strength, speed, and agility. Integrated AI support, energy shielding, and modular components for customization.
   - **Variants:** Multiple variants exist for different combat roles, such as the Recon, Scout, and Defender sets.

2. **Mjolnir GEN3 Armor**
   - **Features:** Further enhancements over GEN2, with improved energy shielding, better mobility, and advanced sensors.
   - **Deployment:** Used primarily by SPARTAN-IVs and later generations.

3. **SPI Armor (Semi-Powered Infiltration)**
   - **Features:** Designed for SPARTAN-IIIs, providing stealth capabilities and lighter protection compared to Mjolnir armor. Enhanced mobility and integrated camouflage systems.

#### Weapons

1. **MA5 Series Assault Rifles**
   - **Overview:** Standard-issue weapons for SPARTANs, reliable and versatile.
   - **Variants:** Includes the MA5B and MA5C, with different specifications for range, accuracy, and fire rate.

2. **M6 Series Pistols**
   - **Overview:** Powerful handguns with high accuracy and stopping power.
   - **Variants:** M6D and M6G are commonly used by SPARTANs.

3. **BR55 Battle Rifle**
   - **Overview:** A burst-fire weapon favored for its precision and effectiveness at medium range.
   - **Features:** Equipped with a scope for improved accuracy.

4. **SRS99 Sniper Rifle**
   - **Overview:** A high-powered, long-range weapon used for precision targeting.
   - **Features:** Equipped with a high-magnification scope and armor-piercing rounds.

5. **M7 SMG**
   - **Overview:** A compact, fully automatic weapon suitable for close-quarters combat.
   - **Features:** High rate of fire and ease of use in tight spaces.

6. **Spartan Laser (M6 G/GNR)**
   - **Overview:** A directed-energy weapon capable of destroying vehicles and heavily armored targets.
   - **Features:** High damage output but requires charging before firing.

7. **Energy Sword**
   - **Overview:** A Covenant melee weapon highly effective in close combat.
   - **Features:** Plasma blade capable of cutting through most materials.

8. **Gravity Hammer**
   - **Overview:** A melee weapon used by the Brutes, capable of generating shockwaves.
   - **Features:** Extremely powerful and can knock back or destroy multiple enemies at once.

### Summary

During the Human-Covenant War, humanity had several generations of SPARTANs, with the SPARTAN-II and SPARTAN-III programs producing the most notable super-soldiers. Post-war, the SPARTAN-IV program expanded the number of SPARTANs significantly.

The latest armor systems, such as the Mjolnir GEN2 and GEN3, provide advanced protection, mobility, and support for the SPARTANs, while a variety of powerful weapons ensure they are well-equipped for any combat scenario.

A team battle between SPARTANs from the Halo universe and Space Marines from the Warhammer 40k universe would be a clash of two elite warrior groups, each with their own unique strengths, technology, and combat strategies. Here’s a detailed comparison to analyze who might emerge victorious in such a confrontation:

### SPARTANs (Halo)

#### Strengths
1. **Training and Tactics:** SPARTANs are highly trained from a young age and are exceptional in both individual combat skills and team-based tactics. They are trained to be adaptable, resourceful, and highly coordinated in combat scenarios.
2. **Augmentations:** SPARTANs undergo rigorous physical and mental augmentations, making them superhumanly strong, fast, and resilient.
3. **Armor Technology:** Mjolnir Powered Assault Armor provides energy shields, enhanced strength and agility, advanced targeting systems, and environmental protection.
4. **Integrated AI:** Some SPARTANs, like Master Chief, have integrated AI (such as Cortana) that provide real-time tactical analysis, hacking capabilities, and battlefield intelligence.
5. **Versatile Weaponry:** SPARTANs have access to a wide range of human and Covenant weaponry, allowing them to adapt to various combat situations.

#### Notable SPARTAN Teams
- **Blue Team:** Led by Master Chief, known for exceptional coordination and effectiveness.
- **Noble Team:** Known for their heroism during the Fall of Reach, demonstrating remarkable combat skills and teamwork.

### Space Marines (Warhammer 40k)

#### Strengths
1. **Genetic Enhancements:** Space Marines undergo extensive genetic modification, giving them superhuman strength, endurance, and resilience. They are significantly larger and more physically imposing than SPARTANs.
2. **Power Armor:** Space Marine Power Armor is heavily fortified, provides enhanced strength, and is equipped with life support systems, advanced sensors, and various wargear options.
3. **Devastating Weaponry:** Space Marines wield powerful weapons like Bolters (which fire explosive rounds), Chainswords, Plasma Guns, and heavy weapons such as Lascannons and Melta Guns.
4. **Psychic Abilities:** Some Space Marines, known as Librarians, have psychic powers that can be used for offense, defense, and battlefield manipulation.
5. **Combat Experience:** Space Marines have centuries or even millennia of combat experience, and their doctrines are based on both ancient wisdom and relentless training.

#### Notable Space Marine Chapters
- **Ultramarines:** Known for their adherence to the Codex Astartes and tactical prowess.
- **Blood Angels:** Renowned for their ferocity in combat and their unique abilities.
- **Space Wolves:** Known for their savagery and exceptional close combat skills.

### Scenario Analysis

#### Physical Attributes and Armor
- **Strength and Durability:** Space Marines generally have the edge in raw physical strength and durability due to their extensive genetic modifications and heavily fortified Power Armor.
- **Agility and Reflexes:** SPARTANs might have an advantage in terms of agility and reflexes, thanks to their augmentations and the advanced capabilities of the Mjolnir armor.

#### Equipment and Technology
- **Weapons:** Space Marine weapons are typically more powerful and designed for high-impact damage, capable of devastating effects. However, SPARTANs have versatile weaponry, including energy shields and advanced targeting systems.
- **Armor:** Both have incredibly advanced armor systems. Space Marine Power Armor offers substantial protection and strength augmentation, while Mjolnir armor provides enhanced agility, shielding, and tactical advantages with AI integration.

#### Tactics and Teamwork
- **Training and Coordination:** Both SPARTANs and Space Marines are elite soldiers with rigorous training. SPARTANs are known for their tactical flexibility and ability to adapt to various combat situations. Space Marines have a more rigid strategic doctrine but are relentless and highly disciplined in their approach.
- **Psychic Abilities:** Space Marines’ Librarians can turn the tide with their psychic powers, providing an unpredictable and potent advantage.

### Possible Outcomes

1. **Open Battlefield:** In a straightforward, open-field battle, Space Marines might have an edge due to their sheer physical power, durability, and devastating weaponry. Their experience and psychic abilities could also play a significant role.
2. **Urban or Complex Terrain:** In environments requiring agility and adaptability, SPARTANs could leverage their superior reflexes, advanced tactical systems, and integrated AI to outmaneuver and outthink Space Marines.
3. **Team Coordination:** Both sides excel in teamwork, but SPARTANs’ adaptability and real-time AI support might give them a slight advantage in dynamic and rapidly changing scenarios.

### Summary
- **Space Marines:** Hold the edge in raw power, durability, heavy weaponry, and psychic abilities. Their combat experience and relentless tactics make them formidable opponents.
- **SPARTANs:** Excel in agility, reflexes, advanced technology, and tactical flexibility. Their integrated AI and versatile weaponry allow them to adapt to various combat situations effectively.

The outcome of such a battle would likely depend heavily on the specific circumstances and terrain. In many scenarios, Space Marines' sheer power and durability could give them the upper hand, while in more dynamic and complex environments, SPARTANs' adaptability and advanced technology could turn the tide in their favor.








Got it. Let's create a plan to integrate the synchronization progress into the toaster component. Here's the detailed step-by-step plan for implementing this feature:

1. **Add progress and totalModels Props to `SpinnerToast.vue`:**
    - Modify the `SpinnerToast.vue` component to accept `progress` and `totalModels` props.
    - Update the template to display the synchronization progress.

2. **Modify `Toaster.vue` to Pass Progress Data:**
    - Ensure the `Toaster.vue` component can pass the `progress` and `totalModels` props to `SpinnerToast`.

3. **Update `toastService` to Handle Synchronization Progress:**
    - Modify the `toastService` to include `progress` and `totalModels` in the toast properties when creating spinner toasts.

4. **Modify `Sync.vue` to Emit Progress Updates:**
    - Update the `Sync.vue` component to emit progress updates via the `toastService`.

### Step 1: Update `SpinnerToast.vue`

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
    <div class="progress">
      {{ progress }} / {{ totalModels }} models are done. <!-- Added progress display -->
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
    progress: number;      // Added new prop for progress
    totalModels: number;   // Added new prop for total models
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    progress: 0,          // Default value for progress
    totalModels: 0        // Default value for total models
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

.progress {
  font-size: 14px;
  color: var(--toaster-progress-text-color); // Style for progress text
  margin-top: 10px;
}
</style>
```

### Step 2: Modify `Toaster.vue`

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
      v-bind="{ id: toast.id, class: 'toast', ...toast.properties, progress: toast.properties.progress, totalModels: toast.properties.totalModels }" <!-- Added progress and totalModels binding -->
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

### Step 3: Update `toastService.ts`

```ts
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
  public displaySpinner(id: symbol, header: string, progress: number, totalModels: number): symbol { // Updated method to accept progress and totalModels
    return this.displayWithId(id, ToastType.Spinner, { title: header, progress, totalModels }); // Pass progress and totalModels to props
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
```

### Step 4: Modify `Sync.vue`

Ensure the `Sync.vue` component can update the toast with progress information.

```vue
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
          <div v-else-if="isSyncSuccessfulDialog" class="successful">
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

import { toastService } from '@/services/toast.js'; // Import toastService

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

  // Emit progress update to the toastService
  const toastId = Symbol('sync-toast');
  const totalModels = props.status?.allModels.length || 0;
  toastService.displaySpinner(toastId, 'Synchronization requested', 0, totalModels);

  const interval = setInterval(() => {
    const progress = props.status?.syncedModels.length || 0;
    toastService.displaySpinner(toastId, 'Synchronization requested', progress, totalModels);
    
    if (progress >= totalModels) {
      clearInterval(interval);
      toastService.dismiss(toastId);
    }
  }, 1000);
}

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
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
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
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }
}

.successful {
  display: flex;
  flex-direction: column;
  align-items: center;

  ul {
    list-style: none;
    padding: 0;
    margin: 0;

    li {
      display: flex;
      align-items: center;
      margin-bottom: var(--base-spacing-1);

      .icon {
        margin-right: var(--base-spacing-1);
      }
    }
  }
}
</style>
```

### Summary of Changes with Comments

1. **`SpinnerToast.vue`:** Added `progress` and `totalModels` props and updated the template to display synchronization progress.
2. **`Toaster.vue`:** Modified the `v-bind` to pass `progress` and `totalModels` to the `SpinnerToast` component.
3. **`toastService.ts`:** Updated the `displaySpinner` method to accept `progress` and `totalModels` and pass them as properties.
4. **`Sync.vue`:** Added logic to emit progress updates to the `toastService`.

By following this plan, you'll be able to display synchronization progress in the toaster component, reflecting the real-time synchronization status to other users in the same workspace. If you encounter any issues or need further assistance, let me know!



-message">.</p>
            </div>

            <div v-if="status?.currentModelName && isSyncUnsuccessfulDialog" class="failed-model">
              <app-icon name="sync_error" />
              <p class="lead dialog-content-message">{{ status.currentModelName }}</p>
            </div>
          </div>
          <div v-else-if="isSyncSuccessfulDialog" class="successful">
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

import { toastService } from '@/services/toast.js'; // Import toastService

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

  // Emit progress update to the toastService
  const toastId = Symbol('sync-toast');  // Create a unique ID for the toast
  const totalModels = props.status?.allModels.length || 0;  // Get total number of models
  toastService.displaySpinner(toastId, 'Synchronization requested', 0, totalModels);  // Initialize the spinner toast

  const interval = setInterval(() => {  // Set up interval to update progress
    const progress = props.status?.syncedModels.length || 0;  // Get current progress
    toastService.displaySpinner(toastId, 'Synchronization requested', progress, totalModels);  // Update spinner toast
    
    if (progress >= totalModels) {  // Check if synchronization is complete
      clearInterval(interval);  // Clear interval
      toastService.dismiss(toastId);  // Dismiss the toast
    }
  }, 1000);  // Interval time set to 1 second
}

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
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
    fill: var(--success);
  }
}

.unsuccessful {
  display: flex;
  flex-direction: column;
  align-items: center;

  .unsuccessful-message {
    display: flex;

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }
}

.successful {
  display: flex;
  flex-direction: column;
  align-items: center;

  ul {
    list-style: none;
    padding: 0;
    margin: 0;

    li {
      display: flex;
      align-items: center;
      margin-bottom: var(--base-spacing-1);

      .icon {
        margin-right: var(--base-spacing-1);
      }
    }
  }
}
</style>
```

With these highlighted changes, you should be able to integrate the synchronization progress into the toaster component

i18n

Got it. I'll ensure all bare texts are properly localized using `i18n`. Here are the revised changes with `i18n` integration for the progress display and other necessary texts.

### `SpinnerToast.vue`

Updated to use `i18n` for localization.

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
    <div class="progress">
      {{ progress }} / {{ totalModels }} {{ t('sync.modelsDone') }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted } from 'vue';
import { useI18n } from 'vue-i18n';

import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';

import { ToastEvent, toastService } from '@/services/toast.js';

defineOptions({ name: 'SpinnerToast' });

const props = withDefaults(
  defineProps<{
    id: symbol;
    title: string;
    titleVars?: Record<string, unknown>;
    dismissable?: boolean;
    progress: number;
    totalModels: number;
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    progress: 0,
    totalModels: 0
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

.progress {
  font-size: 14px;
  color: var(--toaster-progress-text-color);
  margin-top: 10px;
}
</style>
```

### `toast.ts`

Updated to use `i18n` for the spinner title.

```ts
import { DefineComponent } from 'vue';
import mitt, { Emitter, Handler } from 'mitt';
import { LoggingService } from '@ebitoolmx/logging-service/console';
import { loggingService as commonLoggingService } from './logging.js';
import { useI18n } from 'vue-i18n';

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
  public displaySpinner(id: symbol, header: string, progress: number, totalModels: number): symbol {
    return this.displayWithId(id, ToastType.Spinner, { title: header, progress, totalModels });
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
    this



.events.emit(ToastEvent.CloseAll);
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
```

### `Sync.vue`

Updated to use `i18n` for the spinner title and progress message.

```vue
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
          <div v-else-if="isSyncSuccessfulDialog" class="successful">
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
import { useI18n, PickupPaths } from 'vue-i18n';

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

import { toastService } from '@/services/toast.js';

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
  if (!email) return

;

  productsStore.syncProduct({
    productId: productsStore.activeProductId,
    userId: email,
    editorType: editorStore.editorType
  });

  // Emit progress update to the toastService
  const toastId = Symbol('sync-toast');
  const totalModels = props.status?.allModels.length || 0;
  toastService.displaySpinner(toastId, t('sync.spinnerTitle'), 0, totalModels);

  const interval = setInterval(() => {
    const progress = props.status?.syncedModels.length || 0;
    toastService.displaySpinner(toastId, t('sync.spinnerTitle'), progress, totalModels);
    
    if (progress >= totalModels) {
      clearInterval(interval);
      toastService.dismiss(toastId);
    }
  }, 1000);
}

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
      margin-right: var(--base-spacing-1);
    }
  }

  .syncing {
    color: var(--font-color);
    fill: var(--font-color);
  }

  .synced {
    color: var(--success);
    fill: var(--success);
  }
}

.unsuccessful {
  display: flex;
  flex-direction: column;
  align-items: center;

  .unsuccessful-message {
    display: flex;

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
    }
  }

  .failed-model {
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
    }
  }
}

.successful {
  display: flex;
  flex-direction: column;
  align-items: center;

  ul {
    list-style: none;
    padding: 0;
    margin: 0;

    li {
      display: flex;
      align-items: center;
      margin-bottom: var(--base-spacing-1);

      .icon {
        margin-right: var(--base-spacing-1);
      }
    }
  }
}
</style>
```

### Add Translations

Make sure to add the necessary translation keys to your translation files, for example:

**en.json**:
```json
{
  "common": {
    "dismiss": "Dismiss",
    "start": "Start",
    "close": "Close",
    "helpSupport": "Help & Support",
    "link": "https://support.example.com"
  },
  "sync": {
    "modelsDone": "models are done",
    "spinnerTitle": "Synchronization requested"
  }
}
```

With these updates, all bare texts are now localized using `i18n`. If you need any further adjustments or help, let me know!

