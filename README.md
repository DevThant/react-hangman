To ensure the layout of your error message and help desk link match your design, you need to refine the CSS and potentially adjust the HTML structure slightly. The key is to ensure the message text and the link are styled properly and kept on the same line or appropriate lines.

Here's the updated code for `Sync.vue` to match the design closely:

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

              <p class="lead dialog-content-message">
                {{ t('sync.modal.message.contact') }}
                <a
                  :href="t('common.link')"
                  rel="noopener noreferrer"
                  target="_blank"
                  class="lead dialog-content-message"
                >
                  {{ t('common.helpSupport') }}
                </a>.
              </p>
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

    a {
      color: var(--link-color);
    }

    p {
      padding-right: var(--base-spacing-1);
      margin: 0;
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
    display: flex;
    align-items: center;
    color: var(--danger);
    fill: var(--danger);

    p {
      padding-left: var(--base-spacing-1);
      margin: 0;
    }
  }
}
</style>
```

In this updated code:
1. The `.unsuccessful-message` class now has `flex-direction: column` and `align-items: center` to ensure the text and the link are displayed properly on separate lines but still centered.
2. Adjustments have been made to the `p` elements within the `.unsuccessful-message` to ensure there is no extra margin that might break the lines.
3. Updated the `message` in `computed` properties.

This should align the elements correctly and keep the text and link properly formatted.



-------
It looks like there is an extra message being displayed. To address this, we should examine the template and localization strings to ensure there is no duplicate content. Let's update your template and localization strings accordingly.

Here's the updated template and localization:

### Template (Sync.vue)
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

.unsuccess

ful {
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
    flex-direction: row;
    justify-content: center;
    align-items: center;

    p {
      margin-right: var(--base-spacing-1);
    }

    a {
      color: var(--link-color);
    }
  }
}
</style>
```

### Localization (en.ts)
```typescript
export default {
  sync: {
    modal: {
      title: 'Synchronisation',
      header: {
        requested: 'Synchronisation Requested',
        initialising: 'Initialising Synchronisation',
        processing: 'Processing Synchronisation',
        successful: 'Synchronisation Successful',
        initialisingFailed: 'Initialisation Failed',
        unsuccessful: 'Synchronisation Unsuccessful',
        offline: 'Network Offline'
      },
      message: {
        requested: 'Do you wish to start synchronisation?',
        progress: 'Progress',
        contact: 'If the issue remains, contact the',
        unsuccessful: 'A model failed to synchronise.',
        offline:
          'The network connection is lost. Your work will remain.\nPlease check your network connection to be able to continue working.',
        modelFailed: 'The {modelName} model couldn’t be initialised.'
      }
    }
  },
  common: {
    close: 'Close',
    helpSupport: 'help desk',
    link: 'https://helpdesk.example.com'
  }
}
```

This update ensures the "contact" message and the "help desk" link are always together and aligned properly. It also adds the necessary styles to ensure proper alignment and display. The extra message "The model couldn’t be initialised." seems to be coming from the `sync.modal.message.unsuccessful` localization string. Make sure this string is only used where appropriate.


------
To ensure that the contact message and the help desk link always appear together on the same line, you can adjust the structure of the template and CSS slightly. The goal is to group the contact message and the help desk link within the same parent container.

Here is the updated code for Sync.vue:

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


-----

To ensure that the contact message and the help desk link always appear together on the same line, you can adjust the structure of the template and CSS slightly. The goal is to group the contact message and the help desk link within the same parent container.

Here is the updated code for Sync.vue:

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
