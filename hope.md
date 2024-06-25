I see the issue you're facing with the scroll behavior in your application. It seems that the main problem is that the scroll events are propagating to parent elements, which is causing the main `SplitView` to scroll instead of the `SidePanel` or the dropdown within it.

To address this issue, you need to ensure that when the user is interacting with a specific scrollable area, the scroll events are contained within that area and do not propagate to parent elements. Here are the steps you can take to resolve this:

### Step 1: Prevent Scroll Propagation

You need to stop the scroll events from propagating to the parent components. This can be done by adding an event listener to the scrollable elements and calling `event.stopPropagation()`.

### Step 2: Handle Scroll Events in Child Components

Ensure that scroll events in the child components (like the dropdown) are handled properly without affecting the parent components.

### Example Implementation:

Let's update your `SidePanel.vue` and dropdown components to prevent scroll propagation.

#### SidePanel.vue

```vue
<template>
  <aside class="side-panel" data-testid="sidePanel" :data-expanded="shouldShowPanel">
    <section v-show="shouldShowPanel" data-testid="sidePanel-tray" class="panel-tray" @scroll.stop>
      <keep-alive v-for="(component, key) of panels" :key="key">
        <component :is="component" v-if="key === editorStore.sidePanel" />
      </keep-alive>
    </section>
    <section v-show="shouldShowSeparator" class="separator"></section>
    <section v-show="shouldShowSecondaryPanel" data-testid="sidePanel-tray" class="panel-tray" @scroll.stop>
      <keep-alive v-for="(component, key) of panels" :key="key">
        <component :is="component" v-if="key === editorStore.secondarySidePanel" />
      </keep-alive>
    </section>
  </aside>
</template>

<script setup lang="ts">
import { computed, onMounted, onBeforeUnmount, type Component } from 'vue';
import { Shortcut } from '@/typings/shortcut.js';
import { EditorType } from '@/typings/store.js';
import {
  Panel,
  isCosPanel,
  isCbssPanel,
  isCevizPanel,
  isReferenceTreePanel,
  isIlsPanel
} from '@/typings/sidePanel.js';
import { eventService, EventType } from '@/services/event.js';
import { useEditorStore } from '@/stores/editor.js';
import { usePreferencesStore } from '@/stores/preferences.js';

defineOptions({ name: 'SidePanel' });

const props = defineProps<{ panels: { [key: string]: Component } }>();

const preferencesStore = usePreferencesStore();
const editorStore = useEditorStore();

const shouldShowPanel = computed<boolean>(() => editorStore.sidePanel !== Panel.Blank);
const shouldShowSecondaryPanel = computed<boolean>(() => editorStore.secondarySidePanel !== Panel.Blank);
const shouldShowSeparator = computed<boolean>(() => shouldShowPanel.value || shouldShowSecondaryPanel.value);

const panelPresentInEditor = computed<boolean>(() => {
  switch (editorStore.editorType) {
    case EditorType.Cos:
      return isCosPanel(editorStore.sidePanel) && isCosPanel(editorStore.lastSidePanelState);
    case EditorType.Ceviz:
      return isCevizPanel(editorStore.sidePanel) && isCevizPanel(editorStore.lastSidePanelState);
    case EditorType.Cbss:
      return isCbssPanel(editorStore.sidePanel) && isCbssPanel(editorStore.lastSidePanelState);
    case EditorType.Ils:
      return isIlsPanel(editorStore.sidePanel) && isIlsPanel(editorStore.lastSidePanelState);
    case EditorType.ReferenceTree:
      return (
        isReferenceTreePanel(editorStore.sidePanel) &&
        isReferenceTreePanel(editorStore.lastSidePanelState)
      );
    default:
      return false;
  }
});

const isSelected = (panel: Panel): boolean => editorStore.sidePanel === panel;

const toggleSelection = ({ panel }: { panel: Panel; options?: Record<string, any> | undefined }) => {
  const newPanel = isSelected(panel) ? Panel.Blank : panel;
  editorStore.setSidePanel(newPanel);
};

const triggerLastPanel = () => {
  if (shouldShowPanel.value) {
    editorStore.setSidePanel(Panel.Blank);
    editorStore.setSecondarySidePanel(Panel.Blank);
  } else if (panelPresentInEditor.value) {
    editorStore.setSidePanel(editorStore.lastSidePanelState);
  }
};

const shortcutHandler = (shortcut: Shortcut) => {
  if (shortcut === Shortcut.ToggleRecentPanel) triggerLastPanel();
};

onMounted(() => {
  if (
    Object.keys(props.panels).includes(preferencesStore.initialSidePanel) &&
    panelPresentInEditor.value
  ) {
    editorStore.setSidePanel(preferencesStore.initialSidePanel);
  } else {
    editorStore.setSidePanel(Panel.Blank);
  }
  eventService.on(EventType.ToggleSidePanel, toggleSelection);
  eventService.on(EventType.Shortcut, shortcutHandler);
});

onBeforeUnmount(() => {
  eventService.off(EventType.ToggleSidePanel, toggleSelection);
  eventService.off(EventType.Shortcut, shortcutHandler);
});
</script>

<style scoped>
aside {
  display: flex;
  flex-direction: column;
  flex-grow: 0;
  background: var(--header-color);
}

.panel-tray {
  flex-grow: 0;
  flex-shrink: 0;
  flex: 1;
  flex-basis: var(--property-panel-width);
  width: var(--property-panel-width);
  overflow-x: hidden;
  overflow-y: auto;
  border-left: var(--header-border-thickness) solid var(--sidepanel-border-color);
  background: var(--header-color);
  z-index: 3;
}

.separator {
  height: 1.5px;
  border-left: var(--header-border-thickness) solid var(--sidepanel-border-color);
  background-color: var(--sidepanel-border-color);
}

::-webkit-scrollbar-track-piece,
::-webkit-scrollbar-corner,
::-webkit-resizer {
  background: var(--header-color);
}

::-webkit-scrollbar-thumb {
  border: 5px solid var(--header-color);
}
</style>
```

#### Dropdown Component

Ensure you prevent scroll propagation in your dropdown component as well.

```vue
<template>
  <div class="search-property-wrapper" @click="openDropdown">
    <app-input
      id="search-input"
      v-model="searchString"
      raised
      :alt-style="altStyle"
      type="text"
      :placeholder="inputPlaceholder"
      autocomplete="off"
      icon-label="search"
      class="search-input"
      @click.stop="openDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div v-if="opened" ref="searchDropdown" tabindex="0" class="results" @mousedown="focusDropdown" @scroll.stop>
      <ol class="scrollable" @scroll.stop>
        <template
          v-if="
            matchedResultsFilter(searchString, filteredAvailableValues).length ||
            matchedResultsFilter(searchString, selectedValues).length
          "
        >
          <template
            v-if="matchedResultsFilter(searchString, filteredAvailableValues).length && enabled"
          >
            <li class="small">{{ availableLabel }}:</li>
            <li
              v-for="value of matchedResultsFilter(searchString, filteredAvailableValues)"
              :key="parseSearchValue(value)"
              :title="parseSearchValue(value)"
              class="selectable"
              @click="select(value)"
            >
              <slot name="availableValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
            </li>
          </template>
          <template v-if="matchedResultsFilter(searchString, selectedValues).length">
            <li class="small">{{ selectedLabel }}:</li>
            <li
              v-for="value of matchedResultsFilter(searchString, selectedValues)"
              :key="parseSearchValue(value)"
              class="selectable"
              :title="parseSearchValue(value)"
            >
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
            </li>
          </template>
        </template>
        <li v-else-if="!noMatchFound">{{ t('properties.noAvailableValues') }}</li>
        <li v-if="noMatchFound">{{ t('properties.noMatch') }}</li>
      </ol>
    </div>
  </div>
</template>

<script setup lang="ts" generic="T extends MinimalSearchValue">
import { computed, ref, onMounted, onBeforeUnmount, watch } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
import { eventService, EventType } from '@/services/event.js';
import { Shortcut } from '@/typings/shortcut.js';
import {
  matchedResultsFilter,
  MinimalSearchValue,
  parseSearchValue
} from '@/components/common/search/helpers/match.js';

defineOptions({ name: 'SearchProperties' });

const props = withDefaults(
  defineProps<{
    modelValue?: string;
    altStyle?: boolean;
    selectedValues?: T[];
    availableValues?: T[];
    enabled?: boolean;
    inputPlaceholder?: string;
    labelKey?: string;
  }>(),
 

 {
    modelValue: '',
    selectedValues: () => [],
    availableValues: () => [],
    inputPlaceholder: 'Search list',
    labelKey: ''
  }
);

const emit = defineEmits<{
  'update:modelValue': [value: string];
  blur: [value: string];
  selected: [value: T];
  clear: [];
  remove: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const wrapper = ref<HTMLElement | null>(null);
const opened = ref(false);

const filteredAvailableValues = computed<T[]>(() => {
  const valueMap = props.selectedValues.map(val => parseSearchValue(val));
  return props.availableValues.filter(value => !valueMap.includes(parseSearchValue(value)));
});

const noMatchFound = computed<boolean>(
  () =>
    !!searchString.value &&
    !matchedResultsFilter(searchString.value, props.selectedValues).length &&
    (!props.enabled || !matchedResultsFilter(searchString.value, props.availableValues).length)
);

const parsedLabelKey = computed<string>(() => (props.labelKey ? ` ${t(props.labelKey)}` : ''));
const availableLabel = computed<string>(() => t('properties.available') + parsedLabelKey.value);
const selectedLabel = computed<string>(() => t('properties.selected') + parsedLabelKey.value);

const clearSearch = () => {
  searchString.value = '';
  emit('clear');
};

const select = (value: T) => {
  searchString.value = parseSearchValue(value);
  emit('selected', value);
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const openDropdown = () => {
  opened.value = true;
};

const closeDropdown = () => {
  opened.value = false;
  emit('blur', searchString.value);
};

const handleClickOutside = (event: MouseEvent) => {
  if (wrapper.value && !wrapper.value.contains(event.target as Node)) {
    closeDropdown();
  }
};

const handleShortcut = (shortcut: Shortcut) => {
  if (shortcut === Shortcut.Escape) {
    closeDropdown();
  }
};

onMounted(() => {
  eventService.on(EventType.Click, handleClickOutside);
  eventService.on(EventType.Shortcut, handleShortcut);
});

onBeforeUnmount(() => {
  eventService.off(EventType.Click, handleClickOutside);
  eventService.off(EventType.Shortcut, handleShortcut);
});

watch(searchString, value => emit('update:modelValue', value));
</script>

<style scoped>
.search-property-wrapper {
  position: relative;
}
.results {
  width: 100%;
  position: absolute;
  z-index: 9;
  box-shadow: var(--search-results-shadow);
  border: 1px solid var(--search-results-border-color);
}
li {
  background: var(--dropdown-color);
  height: 2.5em;
  padding: 0 var(--base-spacing-2);
  display: flex;
  flex-direction: row;
  align-content: center;
  align-items: center;
  user-select: none;
  white-space: nowrap;
}
.scrollable {
  overflow: hidden;
  overflow-y: auto;
  max-height: 30vh;
}
.selectable {
  cursor: pointer;
}
.selectable:hover {
  background: var(--dropdown-hover-color);
}
.search-input {
  height: 40px;
}
.search-input:deep(.field) {
  height: 100%;
}
</style>
```

### Step 3: Testing the Changes

After making these changes, test the scroll behavior again. Place your mouse over the `SidePanel` or the dropdown and ensure that scrolling within these components does not affect the `SplitView`.

By stopping the scroll propagation, you can ensure that the scroll events are contained within the intended component, allowing for independent scrolling in the `SplitView`, `SidePanel`, and dropdown components. If the issue persists, further debugging might be needed to ensure that no other parts of your application are interfering with the scroll behavior.
