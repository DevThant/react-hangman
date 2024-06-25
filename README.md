To integrate the `eventService` for handling events like clicks and keyboard interactions, we need to replace the direct event listeners in the `SearchProperties.vue` component with the event handling provided by `eventService.ts`. 

Here is the updated `SearchProperties.vue` component:

```vue
<template>
  <div ref="wrapper" class="search-property-wrapper" @click="openDropdown">
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
    <div v-if="opened" ref="searchDropdown" tabindex="0" class="results" @mousedown="focusDropdown">
      <ol class="scrollable">
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
const handleEscapeKey = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    closeDropdown();
  }
};

// Use eventService for handling events
onMounted(() => {
  eventService.on(EventType.Click, handleClickOutside);
  eventService.on(EventType.Shortcut, handleEscapeKey);
});
onBeforeUnmount(() => {
  eventService.off(EventType.Click, handleClickOutside);
  eventService.off(EventType.Shortcut, handleEscapeKey);
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

### Explanation
1. **Imports**: `eventService` and `EventType` are imported from `eventService.ts`.
2. **Event Handling**: The `onMounted` and `onBeforeUnmount` lifecycle hooks are used to add and remove event listeners using `eventService`.
3. **Event Listeners**: Instead of directly adding event listeners to `document`, `eventService.on` is used to handle `EventType.Click` and `EventType.Shortcut` events.

By using the `eventService`, the code becomes more modular and consistent with the event handling approach used in other parts of the application.

---------
<template>
  <div ref="wrapper" class="search-property-wrapper" @click="openDropdown">
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
    <div v-if="opened" ref="searchDropdown" tabindex="0" class="results" @mousedown="focusDropdown">
      <ol class="scrollable">
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
const handleEscapeKey = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    closeDropdown();
  }
};

// Use eventService for handling events
onMounted(() => {
  eventService.on(EventType.Click, handleClickOutside);
  eventService.on(EventType.Shortcut, handleEscapeKey);
});
onBeforeUnmount(() => {
  eventService.off(EventType.Click, handleClickOutside);
  eventService.off(EventType.Shortcut, handleEscapeKey);
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
-----
I apologize for that oversight. Let's properly update the implementation to ensure that the track items are removed correctly from the track and update the state accordingly. Here's the corrected approach:

1. **Ensure Proper Injection of GoJS Diagram Instance**: Make sure the GoJS diagram instance is properly injected and accessible.
2. **Remove Track Item from Track**: Correctly handle the removal of the track item from both the state and the GoJS diagram model.

Here's the updated `TrackItemProperty.vue`:

```vue
<template>
  <app-property-container
    :id="id"
    v-bind="$attrs"
    :full-width="true"
    :editing="isEditing"
    :editable="editorStore.isEditMode"
    icon="picker"
    :title="t('common.edit')"
    :value-size="value.length || 0"
    property-type="trackItemsReference"
    @toggle-editing="toggleEdit"
  >
    <template #label-row-end>
      <div class="label-row-end">
        <app-icon-button
          :name="showAll ? 'chevron-up' : 'chevron-down'"
          :title="t('common.all')"
          @click="toggleShowAll"
        />
      </div>
    </template>
    <div class="element">
      <div class="button-row">
        <app-indicator
          v-if="!isEmpty()"
          class="inline"
          :highlighted="isAllValuesHighlighted()"
          :highlightable="true"
          :type="IndicatorType.Main"
          @click="highlight(value)"
        >
          {{ t('common.all') }}
        </app-indicator>
        <app-indicator
          class="inline"
          :highlightable="true"
          :highlighted="isSplitViewSelected()"
          @click="togglePositionsView()"
        >
          {{ t('common.positions') }}
        </app-indicator>
      </div>
      <div v-if="showAll">
        <app-search-properties
          :available-values="availableValues"
          :selected-values="value"
          :enabled="!$attrs.readonly"
          :input-placeholder="t('properties.search')"
          :label-key="'properties.labels.' + id"
          @selected="addValue"
        >
          <template #selectedValue="selectedValue">
            <app-deletable-indicator
              :value="selectedValue"
              :is-editing="isEditing"
              @delete="deleteValue(selectedValue)"
              @select="selectItemGroup(selectedValue)"
              @highlight="highlight([selectedValue])"
            />
          </template>
        </app-search-properties>
        <div class="prop-values">
          <div class="prop-value-group">
            <div class="order title">{{ t('properties.order') }}</div>
            <div class="value">{{ t('properties.trackItems') }}</div>
          </div>
          <div
            v-for="(group, index) in groupedTrackItems"
            :key="index"
            :class="{ multiple: group.length > 1 }"
            class="prop-value-group"
          >
            <template v-for="[propValue, i] in group" :key="propValue.displayText">
              <div class="order">{{ i }}</div>
              <div class="handle" />
              <app-deletable-indicator
                class="value"
                :value="propValue"
                :is-editing="isEditing"
                @delete="deleteValue(propValue)"
                @select="selectItemGroup(propValue)"
                @highlight="highlight([propValue])"
              />
            </template>
          </div>
        </div>
      </div>
    </div>
  </app-property-container>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useInjectDiagram } from '@/diagram/composables/useDiagram.js';
import { cbssDiagramKey } from '@/diagram/composables/cbss/useCbss.js';

import { PropertyValue } from '@ebitoolmx/cbss-types';
import { XmiId } from '@ebitoolmx/eclipse-types';
import { extractXmiId } from '@ebitoolmx/ebitool-classic-types';
import { isNotEmpty } from '@ebitoolmx/predicates';

import { ToolType, ToolItemCategory } from '@/typings/tools.js';
import { SplitView } from '@/typings/splitView.js';
import { IndicatorType } from '@/typings/indicator.js';
import { PlainSelection } from '@/typings/selection/PlainSelection.js';
import { PlainHighlight } from '@/typings/highlight/PlainHighlight.js';

import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import AppIndicator from '@/components/common/indicator/Indicator.vue';
import AppPropertyContainer from '@/components/common/sidePanelElements/PropertyContainer.vue';
import AppSearchProperties from '@/components/common/search/SearchProperties.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';

import { eventService, EventType } from '@/services/event.js';

import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useDiagramStore } from '@/stores/diagram.js';
import { useEditorStore } from '@/stores/editor.js';
import { isMXObjectIdentifier } from '@/typings/selection';

defineOptions({ name: 'TrackItemProperty' });

const props = withDefaults(
  defineProps<{
    id: string;
    value?: PropertyValue[];
    availableValues?: PropertyValue[];
    category?: string;
  }>(),
  {
    value: () => [],
    availableValues: () => [],
    category: 'Unknown'
  }
);

const emit = defineEmits<{ change: [xmiIds: XmiId[]] }>();

const { t } = useI18n();
const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const diagramStore = useDiagramStore();
const gojsDiagram = useInjectDiagram(cbssDiagramKey);

const showAll = ref<boolean>(false);

const isEditing = computed<boolean>(
  () =>
    diagramStore.selectedToolItem?.category === props.category &&
    diagramStore.selectedToolItem.toolType === ToolType.PickerTool
);

const groupedTrackItems = computed<Array<Array<[PropertyValue, number]>>(() => {
  const groupedByMileage = props.value.reduce(
    (acc: Record<string, Array<[PropertyValue, number]>>, item: PropertyValue, i: number) => {
      const domainObject = infrastructureStore.getTrackItem(item.objectIdentifier.id);
      const mileage = domainObject?.domain.position?.mileage;
      if (mileage && !acc[mileage]) {
        acc[mileage] = [];
      }
      if (mileage && acc[mileage]) {
        acc[mileage].push([item, i + 1]);
      }
      return acc;
    },
    {}
  );

  const formattedArray = Object.values(groupedByMileage);
  return formattedArray;
});

const currentIds = computed<string[]>(() => props.value.map(p => extractXmiId(p.object.reference)));

function getNodesToHighlight(propValues: PropertyValue[]): string[] {
  const nodes = propValues.map(value => value.objectIdentifier.id);
  return nodes.filter(isNotEmpty);
}

function isAllValuesHighlighted(): boolean {
  if (editorStore.highlighted.isEmpty()) return false;
  const nodesToHighlight = getNodesToHighlight(props.value).filter(Boolean);
  return nodesToHighlight.every(node => editorStore.highlighted.expandedIds.includes(node));
}

function isHighlighted(id: PropertyValue): boolean {
  const [nodeToHighlight] = getNodesToHighlight([id]);
  return editorStore.highlighted.includes(nodeToHighlight);
}

const highlight = (propValues: PropertyValue[]): void => {
  const nodesToHighlight = getNodesToHighlight(propValues);
  editorStore.toggleHighlighted(new PlainHighlight(nodesToHighlight));
  eventService.emit(EventType.CenterHighlighted);

  for (const propValue of propValues) {
    const layer = propValue.object.eClass;
    diagramStore.setLayerVisibility([{ layer, visible: true }]);
  }
};

const removeTrackItemFromTrack = (xmiId: XmiId): void => {
  const trackItem = infrastructureStore.getTrackItem(xmiId);
  if (trackItem) {
    const updatedTrackItem = {
      ...trackItem,
      domain: {
        ...trackItem.domain,
        track: null // Remove association with the track
      }
    };

    // Update the track item in the store
    infrastructureStore.updateData([updatedTrackItem]);

    // Update the GoJS diagram to reflect the change
    gojsDiagram.diagram.model.setDataProperty(gojsDiagram.diagram.findNodeForKey(trackItem.xmiId), 'track', null);
  }
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );

  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );

  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));

  removeTrackItemFromTrack(xmiId);
};

const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(new PlainHighlight([...editorStore.highlighted.expandedIds, xmiId]));
  emit('change', [...currentIds.value, xmiId]);
};

const addValue = (propValue: PropertyValue): void => {


  const includeXmiId = extractXmiId(propValue.object.reference);
  if (!currentIds.value.includes(includeXmiId)) addSelected(includeXmiId);
};

const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
};

const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool'
    });

    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);
    }
    return;
  }

  diagramStore.selectToolItem(null);
};

const isEmpty = (): boolean => currentIds.value.length === 0;

const toggleShowAll = (): void => {
  showAll.value = !showAll.value;
};

const isSplitViewSelected = (): boolean => editorStore.isSplitViewShown(SplitView.Positions);

const togglePositionsView = (): void => {
  editorStore.toggleSplitView(SplitView.Positions);
};

const handleSidePanelToggle = (): void => {
  if (isEditing.value) {
    toggleEdit();
  }
};

const selectItemGroup = (propValue: PropertyValue): void => {
  const selection = new PlainSelection([propValue.objectIdentifier]);
  editorStore.setSelected(selection);

  const nodesToHighlight = getNodesToHighlight([propValue]);
  editorStore.setHighlighted(new PlainHighlight(nodesToHighlight));

  if (isHighlighted(propValue)) {
    editorStore.setHighlighted(
      new PlainHighlight(
        editorStore.highlighted.filter((id: string) => id !== propValue.objectIdentifier.id)
      )
    );
  }
};

onMounted(() => eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle));

onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

<style scoped>
.button-row {
  display: flex;
  padding-bottom: var(--base-spacing-2);

  & > * {
    margin-right: var(--base-spacing-2);
  }
}

.element {
  margin-top: var(--base-spacing-2);
}

.label-row-end {
  flex-grow: 1;
  display: flex;
  justify-content: flex-end;

  & svg {
    width: 2em;
    height: 2em;
  }
}

.inline {
  width: fit-content;
}

.prop-values {
  display: grid;
  grid-template-columns: [order] 2em [handle] 3em [value] auto;
  column-gap: 1em;
  margin-top: 1.5em;
}

.prop-value-group {
  display: grid;
  grid-column: 1 / span 3;
  grid-template-columns: subgrid;
}

.handle {
  grid-column: handle;
}

.multiple {
  & .handle {
    border-right: 1px solid var(--track-item-order-group-line-color);
  }

  &:hover {
    background: var(--dropdown-item-hover-color);

    & .value {
      background: var(--dropdown-item-hover-color);
    }

    & .handle {
      border-right: 1px solid var(--track-item-order-group-line-hover);
    }
  }

  &::before {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }
  &::after {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }

  & > * {
    &:nth-child(-n + 3) {
      margin-top: 0;
    }

    &:last-child {
      margin-bottom: 0;
    }
  }
}

.order {
  display: flex;
  align-items: center;
  grid-column: order;
  margin-left: 1em;
  color: var(--track-item-order-font-color);

  &.title {
    margin-left: 0;
  }
}

.value {
  grid-column: value;
  margin: var(--base-spacing-1) 0 var(--base-spacing-1) 0;
}
</style>
```

**Key Changes:**
1. **Properly Injected `useInjectDiagram`**: Make sure the correct instance of the diagram is obtained.
2. **Properly Updated `removeTrackItemFromTrack`**: This function now properly updates both the state and the GoJS diagram model to reflect the removal of the track item. 

Make sure that `useInjectDiagram` composable is correctly imported and used to obtain the GoJS diagram instance. This ensures that the updates are properly reflected in the diagram. If the GoJS diagram instance is not injected correctly, you might need to ensure the correct setup in your Vue component structure and ensure the instance is provided and injected as expected.
----
To address the issue where the track items are not being properly removed from the track, let's adjust the implementation to ensure that the track items are removed correctly both from the state and the UI. The issue might be due to the way the data is being updated and reflected in the GoJS diagram. Here is an updated approach:

1. **Update the Vue component state** to remove the track item.
2. **Update the GoJS diagram model** to reflect the removal.

Here's the updated `TrackItemProperty.vue`:

```vue
<template>
  <app-property-container
    :id="id"
    v-bind="$attrs"
    :full-width="true"
    :editing="isEditing"
    :editable="editorStore.isEditMode"
    icon="picker"
    :title="t('common.edit')"
    :value-size="value.length || 0"
    property-type="trackItemsReference"
    @toggle-editing="toggleEdit"
  >
    <template #label-row-end>
      <div class="label-row-end">
        <app-icon-button
          :name="showAll ? 'chevron-up' : 'chevron-down'"
          :title="t('common.all')"
          @click="toggleShowAll"
        />
      </div>
    </template>
    <div class="element">
      <div class="button-row">
        <app-indicator
          v-if="!isEmpty()"
          class="inline"
          :highlighted="isAllValuesHighlighted()"
          :highlightable="true"
          :type="IndicatorType.Main"
          @click="highlight(value)"
        >
          {{ t('common.all') }}
        </app-indicator>
        <app-indicator
          class="inline"
          :highlightable="true"
          :highlighted="isSplitViewSelected()"
          @click="togglePositionsView()"
        >
          {{ t('common.positions') }}
        </app-indicator>
      </div>
      <div v-if="showAll">
        <app-search-properties
          :available-values="availableValues"
          :selected-values="value"
          :enabled="!$attrs.readonly"
          :input-placeholder="t('properties.search')"
          :label-key="'properties.labels.' + id"
          @selected="addValue"
        >
          <template #selectedValue="selectedValue">
            <app-deletable-indicator
              :value="selectedValue"
              :is-editing="isEditing"
              @delete="deleteValue(selectedValue)"
              @select="selectItemGroup(selectedValue)"
              @highlight="highlight([selectedValue])"
            />
          </template>
        </app-search-properties>
        <div class="prop-values">
          <div class="prop-value-group">
            <div class="order title">{{ t('properties.order') }}</div>
            <div class="value">{{ t('properties.trackItems') }}</div>
          </div>
          <div
            v-for="(group, index) in groupedTrackItems"
            :key="index"
            :class="{ multiple: group.length > 1 }"
            class="prop-value-group"
          >
            <template v-for="[propValue, i] in group" :key="propValue.displayText">
              <div class="order">{{ i }}</div>
              <div class="handle" />
              <app-deletable-indicator
                class="value"
                :value="propValue"
                :is-editing="isEditing"
                @delete="deleteValue(propValue)"
                @select="selectItemGroup(propValue)"
                @highlight="highlight([propValue])"
              />
            </template>
          </div>
        </div>
      </div>
    </div>
  </app-property-container>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useInjectDiagram } from '@/diagram/composables/useDiagram.js';
import { cbssDiagramKey } from '@/diagram/composables/cbss/useCbss.js';

import { PropertyValue } from '@ebitoolmx/cbss-types';
import { XmiId } from '@ebitoolmx/eclipse-types';
import { extractXmiId } from '@ebitoolmx/ebitool-classic-types';
import { isNotEmpty } from '@ebitoolmx/predicates';

import { ToolType, ToolItemCategory } from '@/typings/tools.js';
import { SplitView } from '@/typings/splitView.js';
import { IndicatorType } from '@/typings/indicator.js';
import { PlainSelection } from '@/typings/selection/PlainSelection.js';
import { PlainHighlight } from '@/typings/highlight/PlainHighlight.js';

import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import AppIndicator from '@/components/common/indicator/Indicator.vue';
import AppPropertyContainer from '@/components/common/sidePanelElements/PropertyContainer.vue';
import AppSearchProperties from '@/components/common/search/SearchProperties.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';

import { eventService, EventType } from '@/services/event.js';

import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useDiagramStore } from '@/stores/diagram.js';
import { useEditorStore } from '@/stores/editor.js';
import { isMXObjectIdentifier } from '@/typings/selection';

defineOptions({ name: 'TrackItemProperty' });

const props = withDefaults(
  defineProps<{
    id: string;
    value?: PropertyValue[];
    availableValues?: PropertyValue[];
    category?: string;
  }>(),
  {
    value: () => [],
    availableValues: () => [],
    category: 'Unknown'
  }
);

const emit = defineEmits<{ change: [xmiIds: XmiId[]] }>();

const { t } = useI18n();
const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const diagramStore = useDiagramStore();
const gojsDiagram = useInjectDiagram(cbssDiagramKey);

const showAll = ref<boolean>(false);

const isEditing = computed<boolean>(
  () =>
    diagramStore.selectedToolItem?.category === props.category &&
    diagramStore.selectedToolItem.toolType === ToolType.PickerTool
);

const groupedTrackItems = computed<Array<Array<[PropertyValue, number]>>(() => {
  const groupedByMileage = props.value.reduce(
    (acc: Record<string, Array<[PropertyValue, number]>>, item: PropertyValue, i: number) => {
      const domainObject = infrastructureStore.getTrackItem(item.objectIdentifier.id);
      const mileage = domainObject?.domain.position?.mileage;
      if (mileage && !acc[mileage]) {
        acc[mileage] = [];
      }
      if (mileage && acc[mileage]) {
        acc[mileage].push([item, i + 1]);
      }
      return acc;
    },
    {}
  );

  const formattedArray = Object.values(groupedByMileage);
  return formattedArray;
});

const currentIds = computed<string[]>(() => props.value.map(p => extractXmiId(p.object.reference)));

function getNodesToHighlight(propValues: PropertyValue[]): string[] {
  const nodes = propValues.map(value => value.objectIdentifier.id);
  return nodes.filter(isNotEmpty);
}

function isAllValuesHighlighted(): boolean {
  if (editorStore.highlighted.isEmpty()) return false;
  const nodesToHighlight = getNodesToHighlight(props.value).filter(Boolean);
  return nodesToHighlight.every(node => editorStore.highlighted.expandedIds.includes(node));
}

function isHighlighted(id: PropertyValue): boolean {
  const [nodeToHighlight] = getNodesToHighlight([id]);
  return editorStore.highlighted.includes(nodeToHighlight);
}

const highlight = (propValues: PropertyValue[]): void => {
  const nodesToHighlight = getNodesToHighlight(propValues);
  editorStore.toggleHighlighted(new PlainHighlight(nodesToHighlight));
  eventService.emit(EventType.CenterHighlighted);

  for (const propValue of propValues) {
    const layer = propValue.object.eClass;
    diagramStore.setLayerVisibility([{ layer, visible: true }]);
  }
};

const removeTrackItemFromTrack = (xmiId: XmiId): void => {
  const trackItem = infrastructureStore.getTrackItem(xmiId);
  if (trackItem) {
    const updatedTrackItem = {
      ...trackItem,
      domain: {
        ...trackItem.domain,
        track: null // Remove association with the track
      }
    };

    // Update the track item in the store
    infrastructureStore.updateData([updatedTrackItem]);

    // Update the GoJS diagram to reflect the change
    gojsDiagram.diagram.model.setDataProperty(trackItem, 'track', null);
  }
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );

  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );

  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));

  removeTrackItemFromTrack(xmiId);
};

const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(new PlainHighlight([...editorStore.highlighted.expandedIds, xmiId]));
  emit('change', [...currentIds.value, xmiId]);
};

const addValue = (propValue: PropertyValue): void => {
  const includeXmiId

 = extractXmiId(propValue.object.reference);
  if (!currentIds.value.includes(includeXmiId)) addSelected(includeXmiId);
};

const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
};

const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool'
    });

    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);
    }
    return;
  }

  diagramStore.selectToolItem(null);
};

const isEmpty = (): boolean => currentIds.value.length === 0;

const toggleShowAll = (): void => {
  showAll.value = !showAll.value;
};

const isSplitViewSelected = (): boolean => editorStore.isSplitViewShown(SplitView.Positions);

const togglePositionsView = (): void => {
  editorStore.toggleSplitView(SplitView.Positions);
};

const handleSidePanelToggle = (): void => {
  if (isEditing.value) {
    toggleEdit();
  }
};

const selectItemGroup = (propValue: PropertyValue): void => {
  const selection = new PlainSelection([propValue.objectIdentifier]);
  editorStore.setSelected(selection);

  const nodesToHighlight = getNodesToHighlight([propValue]);
  editorStore.setHighlighted(new PlainHighlight(nodesToHighlight));

  if (isHighlighted(propValue)) {
    editorStore.setHighlighted(
      new PlainHighlight(
        editorStore.highlighted.filter((id: string) => id !== propValue.objectIdentifier.id)
      )
    );
  }
};

onMounted(() => eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle));

onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

<style scoped>
.button-row {
  display: flex;
  padding-bottom: var(--base-spacing-2);

  & > * {
    margin-right: var(--base-spacing-2);
  }
}

.element {
  margin-top: var(--base-spacing-2);
}

.label-row-end {
  flex-grow: 1;
  display: flex;
  justify-content: flex-end;

  & svg {
    width: 2em;
    height: 2em;
  }
}

.inline {
  width: fit-content;
}

.prop-values {
  display: grid;
  grid-template-columns: [order] 2em [handle] 3em [value] auto;
  column-gap: 1em;
  margin-top: 1.5em;
}

.prop-value-group {
  display: grid;
  grid-column: 1 / span 3;
  grid-template-columns: subgrid;
}

.handle {
  grid-column: handle;
}

.multiple {
  & .handle {
    border-right: 1px solid var(--track-item-order-group-line-color);
  }

  &:hover {
    background: var(--dropdown-item-hover-color);

    & .value {
      background: var(--dropdown-item-hover-color);
    }

    & .handle {
      border-right: 1px solid var(--track-item-order-group-line-hover);
    }
  }

  /* Used to make sure borders aren't extending to the edge of the groups */
  &::before {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }
  &::after {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }

  & > * {
    /* Remove margin for top row of each group */
    &:nth-child(-n + 3) {
      margin-top: 0;
    }

    /* Remove margin for last element of each group. Works currently because only the .value elements have margins */
    &:last-child {
      margin-bottom: 0;
    }
  }
}

.order {
  display: flex;
  align-items: center;
  grid-column: order;
  margin-left: 1em;
  color: var(--track-item-order-font-color);

  &.title {
    margin-left: 0;
  }
}

.value {
  grid-column: value;
  margin: var(--base-spacing-1) 0 var(--base-spacing-1) 0;
}
</style>
```

Key changes:
1. **Correct Import for `useInjectDiagram`**: Ensure the correct import statement is used to get the GoJS diagram instance.
2. **Updated `removeTrackItemFromTrack` Function**: This function now properly updates both the Vuex store and the GoJS diagram to reflect the removal of the track item.

Make sure that the `useInjectDiagram` composable is correctly imported and used to obtain the GoJS diagram instance. This ensures that the updates are properly reflected in the diagram.
-----
To address the issue with `gojsDiagram` not being recognized in the `TrackItemProperty.vue`, we need to ensure that the `gojsDiagram` instance is properly imported and used within the `TrackItemProperty.vue` file. Let's update the `TrackItemProperty.vue` to include the necessary imports and adjustments.

Here's the updated implementation for `TrackItemProperty.vue`:

```vue
<template>
  <app-property-container
    :id="id"
    v-bind="$attrs"
    :full-width="true"
    :editing="isEditing"
    :editable="editorStore.isEditMode"
    icon="picker"
    :title="t('common.edit')"
    :value-size="value.length || 0"
    property-type="trackItemsReference"
    @toggle-editing="toggleEdit"
  >
    <template #label-row-end>
      <div class="label-row-end">
        <app-icon-button
          :name="showAll ? 'chevron-up' : 'chevron-down'"
          :title="t('common.all')"
          @click="toggleShowAll"
        />
      </div>
    </template>
    <div class="element">
      <div class="button-row">
        <app-indicator
          v-if="!isEmpty()"
          class="inline"
          :highlighted="isAllValuesHighlighted()"
          :highlightable="true"
          :type="IndicatorType.Main"
          @click="highlight(value)"
        >
          {{ t('common.all') }}
        </app-indicator>
        <app-indicator
          class="inline"
          :highlightable="true"
          :highlighted="isSplitViewSelected()"
          @click="togglePositionsView()"
        >
          {{ t('common.positions') }}
        </app-indicator>
      </div>
      <div v-if="showAll">
        <app-search-properties
          :available-values="availableValues"
          :selected-values="value"
          :enabled="!$attrs.readonly"
          :input-placeholder="t('properties.search')"
          :label-key="'properties.labels.' + id"
          @selected="addValue"
          ><template #selectedValue="selectedValue">
            <app-deletable-indicator
              :value="selectedValue"
              :is-editing="isEditing"
              @delete="deleteValue(selectedValue)"
              @select="selectItemGroup(selectedValue)"
              @highlight="highlight([selectedValue])" /></template
        ></app-search-properties>
        <div class="prop-values">
          <div class="prop-value-group">
            <div class="order title">{{ t('properties.order') }}</div>
            <div class="value">{{ t('properties.trackItems') }}</div>
          </div>
          <div
            v-for="(group, index) in groupedTrackItems"
            :key="index"
            :class="{ multiple: group.length > 1 }"
            class="prop-value-group"
          >
            <template v-for="[propValue, i] in group" :key="propValue.displayText">
              <div class="order">{{ i }}</div>
              <div class="handle" />
              <app-deletable-indicator
                class="value"
                :value="propValue"
                :is-editing="isEditing"
                @delete="deleteValue(propValue)"
                @select="selectItemGroup(propValue)"
                @highlight="highlight([propValue])"
              />
            </template>
          </div>
        </div>
      </div>
    </div>
  </app-property-container>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { useInjectDiagram } from '@/diagram/composables/useDiagram.js'; // Import useInjectDiagram
import { cbssDiagramKey } from '@/diagram/composables/cbss/useCbss.js'; // Import cbssDiagramKey

import { PropertyValue } from '@ebitoolmx/cbss-types';
import { XmiId } from '@ebitoolmx/eclipse-types';
import { extractXmiId } from '@ebitoolmx/ebitool-classic-types';
import { isNotEmpty } from '@ebitoolmx/predicates';

import { ToolType, ToolItemCategory } from '@/typings/tools.js';
import { SplitView } from '@/typings/splitView.js';
import { IndicatorType } from '@/typings/indicator.js';
import { PlainSelection } from '@/typings/selection/PlainSelection.js';
import { PlainHighlight } from '@/typings/highlight/PlainHighlight.js';

import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import AppIndicator from '@/components/common/indicator/Indicator.vue';
import AppPropertyContainer from '@/components/common/sidePanelElements/PropertyContainer.vue';
import AppSearchProperties from '@/components/common/search/SearchProperties.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';

import { eventService, EventType } from '@/services/event.js';

import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useDiagramStore } from '@/stores/diagram.js';
import { useEditorStore } from '@/stores/editor.js';
import { isMXObjectIdentifier } from '@/typings/selection';

defineOptions({ name: 'TrackItemProperty' });

const props = withDefaults(
  defineProps<{
    id: string;
    value?: PropertyValue[];
    availableValues?: PropertyValue[];
    category?: string;
  }>(),
  {
    value: () => [],
    availableValues: () => [],
    category: 'Unknown'
  }
);

const emit = defineEmits<{ change: [xmiIds: XmiId[]] }>();

const { t } = useI18n();
const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const diagramStore = useDiagramStore();
const gojsDiagram = useInjectDiagram(cbssDiagramKey); // Use gojsDiagram

const showAll = ref<boolean>(false);

const isEditing = computed<boolean>(
  () =>
    diagramStore.selectedToolItem?.category === props.category &&
    diagramStore.selectedToolItem.toolType === ToolType.PickerTool
);

const groupedTrackItems = computed<Array<Array<[PropertyValue, number]>>(() => {
  const groupedByMileage = props.value.reduce(
    (acc: Record<string, Array<[PropertyValue, number]>>, item: PropertyValue, i: number) => {
      const domainObject = infrastructureStore.getTrackItem(item.objectIdentifier.id);
      const mileage = domainObject?.domain.position?.mileage;
      if (mileage && !acc[mileage]) {
        acc[mileage] = [];
      }
      if (mileage && acc[mileage]) {
        acc[mileage].push([item, i + 1]);
      }
      return acc;
    },
    {}
  );

  const formattedArray = Object.values(groupedByMileage);
  return formattedArray;
});

const currentIds = computed<string[]>(() => props.value.map(p => extractXmiId(p.object.reference)));

function getNodesToHighlight(propValues: PropertyValue[]): string[] {
  const nodes = propValues.map(value => value.objectIdentifier.id);
  return nodes.filter(isNotEmpty);
}

function isAllValuesHighlighted(): boolean {
  if (editorStore.highlighted.isEmpty()) return false;
  const nodesToHighlight = getNodesToHighlight(props.value).filter(Boolean);
  return nodesToHighlight.every(node => editorStore.highlighted.expandedIds.includes(node));
}

function isHighlighted(id: PropertyValue): boolean {
  const [nodeToHighlight] = getNodesToHighlight([id]);
  return editorStore.highlighted.includes(nodeToHighlight);
}

const highlight = (propValues: PropertyValue[]): void => {
  const nodesToHighlight = getNodesToHighlight(propValues);
  editorStore.toggleHighlighted(new PlainHighlight(nodesToHighlight));
  eventService.emit(EventType.CenterHighlighted);
  for (const propValue of propValues) {
    const layer = propValue.object.eClass;
    diagramStore.setLayerVisibility([{ layer, visible: true }]);
  }
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );

  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};

const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(new PlainHighlight([...editorStore.highlighted.expandedIds, xmiId]));
  emit('change', [...currentIds.value, xmiId]);
};

const addValue = (propValue: PropertyValue): void => {
  const includeXmiId = extractXmiId(propValue.object.reference);
  if (!currentIds.value.includes(includeXmiId)) addSelected(includeXmiId);
};

const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
};

const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool'
    });
    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);


    }
    return;
  }
  diagramStore.selectToolItem(null);
};

const isEmpty = (): boolean => currentIds.value.length === 0;

const toggleShowAll = (): void => {
  showAll.value = !showAll.value;
};

const isSplitViewSelected = (): boolean => editorStore.isSplitViewShown(SplitView.Positions);

const togglePositionsView = (): void => {
  editorStore.toggleSplitView(SplitView.Positions);
};

const handleSidePanelToggle = (): void => {
  if (isEditing.value) {
    toggleEdit();
  }
};

const selectItemGroup = (propValue: PropertyValue): void => {
  const selection = new PlainSelection([propValue.objectIdentifier]);
  editorStore.setSelected(selection);
  const nodesToHighlight = getNodesToHighlight([propValue]);
  editorStore.setHighlighted(new PlainHighlight(nodesToHighlight));
  if (isHighlighted(propValue)) {
    editorStore.setHighlighted(
      new PlainHighlight(
        editorStore.highlighted.filter((id: string) => id !== propValue.objectIdentifier.id)
      )
    );
  }
};

const removeTrackItemFromTrack = (trackItemId: string) => {
  const trackItem = infrastructureStore.getTrackItem(trackItemId);
  if (trackItem) {
    gojsDiagram.diagram.model.setDataProperty(trackItem, 'track', null);
  }
};

onMounted(() => eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle));

onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

<style scoped>
.button-row {
  display: flex;
  padding-bottom: var(--base-spacing-2);

  & > * {
    margin-right: var(--base-spacing-2);
  }
}

.element {
  margin-top: var(--base-spacing-2);
}

.label-row-end {
  flex-grow: 1;
  display: flex;
  justify-content: flex-end;

  & svg {
    width: 2em;
    height: 2em;
  }
}

.inline {
  width: fit-content;
}

.prop-values {
  display: grid;
  grid-template-columns: [order] 2em [handle] 3em [value] auto;
  column-gap: 1em;
  margin-top: 1.5em;
}

.prop-value-group {
  display: grid;
  grid-column: 1 / span 3;
  grid-template-columns: subgrid;
}

.handle {
  grid-column: handle;
}

.multiple {
  & .handle {
    border-right: 1px solid var(--track-item-order-group-line-color);
  }

  &:hover {
    background: var(--dropdown-item-hover-color);

    & .value {
      background: var(--dropdown-item-hover-color);
    }

    & .handle {
      border-right: 1px solid var(--track-item-order-group-line-hover);
    }
  }

  &::before {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }
  &::after {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }

  & > * {
    &:nth-child(-n + 3) {
      margin-top: 0;
    }

    &:last-child {
      margin-bottom: 0;
    }
  }
}

.order {
  display: flex;
  align-items: center;
  grid-column: order;
  margin-left: 1em;
  color: var(--track-item-order-font-color);

  &.title {
    margin-left: 0;
  }
}

.value {
  grid-column: value;
  margin: var(--base-spacing-1) 0 var(--base-spacing-1) 0;
}
</style>
```

In this updated version of `TrackItemProperty.vue`, the `gojsDiagram` instance is injected using `useInjectDiagram` from the appropriate composable, and the key `cbssDiagramKey` is provided. This should resolve the issue with `gojsDiagram` not being recognized.
-----
Sure, I'll provide a full implementation of the new feature where clicking the "x" removes the track item from the `TrackItemProperties` instead of just deselecting it, but does not entirely delete the item from the GoJS diagram.

### Required Steps:
1. **Update `deleteSelected` method in `TrackItemProperty.vue`** to remove the track item from the track items list without deleting it entirely.
2. **Update the state and GoJS diagram** to reflect the changes.

### Implementation

#### **TrackItemProperty.vue**

First, let's modify the `deleteSelected` method in `TrackItemProperty.vue`:

```javascript
const deleteSelected = (xmiId: XmiId): void => {
  // Emit the change event to update the parent component state
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );

  // Remove the item from the highlighted state
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );

  // Update the selected items
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));

  // Call a method to remove the track item from the track
  removeTrackItemFromTrack(xmiId);
};

// New method to remove track item from track
const removeTrackItemFromTrack = (xmiId: XmiId): void => {
  const trackItem = infrastructureStore.getTrackItem(xmiId);
  if (trackItem) {
    const updatedTrackItem = {
      ...trackItem,
      domain: {
        ...trackItem.domain,
        track: null // Remove association with the track
      }
    };

    // Update the track item in the store
    infrastructureStore.updateData([updatedTrackItem]);

    // Update the GoJS diagram to reflect the change
    gojsDiagram.diagram.model.setDataProperty(trackItem, 'track', null);
  }
};
```

### Explanation:
- The `deleteSelected` method now calls `removeTrackItemFromTrack` to handle the removal of the track item from the track.
- `removeTrackItemFromTrack` updates the track item in the store and the GoJS diagram to remove its association with the track.

### Updating Store

Ensure the store (`infrastructure.ts`) has the required methods to handle updating track items:

```javascript
const updateData = (domainObjects: DomainObject[]): void => {
  for (const domainObject of domainObjects) {
    let stateArray: DomainObject[] | null = null;

    if (isTrackItem(domainObject)) stateArray = trackItems.value;
    // ... other conditions

    if (stateArray) {
      const index = stateArray.findIndex(
        (stateItem: DomainObject) => stateItem.xmiId === domainObject.xmiId
      );

      if (index > -1) {
        stateArray[index] = Object.freeze(domainObject);
        eventService.emit(EventType.DomainObjectUpdated, domainObject.xmiId);
      } else {
        stateArray.push(domainObject);
        eventService.emit(EventType.DomainObjectUpdated, domainObject.xmiId);
      }
    }
  }
};
```

### Full Implementation in `TrackItemProperty.vue`

```javascript
<template>
  <!-- ... existing template code ... -->
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { PropertyValue } from '@ebitoolmx/cbss-types';
import { XmiId } from '@ebitoolmx/eclipse-types';
import { extractXmiId } from '@ebitoolmx/ebitool-classic-types';
import { isNotEmpty } from '@ebitoolmx/predicates';
import { useEditorStore } from '@/stores/editor.js';
import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useDiagramStore } from '@/stores/diagram.js';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import AppIndicator from '@/components/common/indicator/Indicator.vue';
import AppPropertyContainer from '@/components/common/sidePanelElements/PropertyContainer.vue';
import AppSearchProperties from '@/components/common/search/SearchProperties.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { eventService, EventType } from '@/services/event.js';
import { PlainSelection } from '@/typings/selection/PlainSelection.js';
import { PlainHighlight } from '@/typings/highlight/PlainHighlight.js';
import { ToolType } from '@/typings/tools.js';
import { SplitView } from '@/typings/splitView.js';

defineOptions({ name: 'TrackItemProperty' });

const props = withDefaults(
  defineProps<{
    id: string;
    value?: PropertyValue[];
    availableValues?: PropertyValue[];
    category?: string;
  }>(),
  {
    value: () => [],
    availableValues: () => [],
    category: 'Unknown'
  }
);

const emit = defineEmits<{ change: [xmiIds: XmiId[]] }>();

const { t } = useI18n();
const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const diagramStore = useDiagramStore();

const showAll = ref<boolean>(false);
const isEditing = computed<boolean>(
  () =>
    diagramStore.selectedToolItem?.category === props.category &&
    diagramStore.selectedToolItem.toolType === ToolType.PickerTool
);
const groupedTrackItems = computed<Array<Array<[PropertyValue, number]>>>(() => {
  const groupedByMileage = props.value.reduce(
    (acc: Record<string, Array<[PropertyValue, number]>>, item: PropertyValue, i: number) => {
      const domainObject = infrastructureStore.getTrackItem(item.objectIdentifier.id);
      const mileage = domainObject?.domain.position?.mileage;
      if (mileage && !acc[mileage]) {
        acc[mileage] = [];
      }
      if (mileage && acc[mileage]) {
        acc[mileage].push([item, i + 1]);
      }
      return acc;
    },
    {}
  );
  const formattedArray = Object.values(groupedByMileage);
  return formattedArray;
});
const currentIds = computed<string[]>(() => props.value.map(p => extractXmiId(p.object.reference)));

function getNodesToHighlight(propValues: PropertyValue[]): string[] {
  const nodes = propValues.map(value => value.objectIdentifier.id);
  return nodes.filter(isNotEmpty);
}

function isAllValuesHighlighted(): boolean {
  if (editorStore.highlighted.isEmpty()) return false;
  const nodesToHighlight = getNodesToHighlight(props.value).filter(Boolean);
  return nodesToHighlight.every(node => editorStore.highlighted.expandedIds.includes(node));
}

function isHighlighted(id: PropertyValue): boolean {
  const [nodeToHighlight] = getNodesToHighlight([id]);
  return editorStore.highlighted.includes(nodeToHighlight);
}

const highlight = (propValues: PropertyValue[]): void => {
  const nodesToHighlight = getNodesToHighlight(propValues);
  editorStore.toggleHighlighted(new PlainHighlight(nodesToHighlight));
  eventService.emit(EventType.CenterHighlighted);
  for (const propValue of propValues) {
    const layer = propValue.object.eClass;
    diagramStore.setLayerVisibility([{ layer, visible: true }]);
  }
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
  removeTrackItemFromTrack(xmiId);
};

const removeTrackItemFromTrack = (xmiId: XmiId): void => {
  const trackItem = infrastructureStore.getTrackItem(xmiId);
  if (trackItem) {
    const updatedTrackItem = {
      ...trackItem,
      domain: {
        ...trackItem.domain,
        track: null // Remove association with the track
      }
    };
    infrastructureStore.updateData([updatedTrackItem]);
    gojsDiagram.diagram.model.setDataProperty(trackItem, 'track', null);
  }
};

const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(new PlainHighlight([...editorStore.highlighted.expandedIds, xmiId]));
  emit('change', [...currentIds.value, xmiId]);
};

const addValue = (propValue: PropertyValue): void => {
  const includeXmiId = extractXmiId(propValue.object.reference);
  if (!currentIds.value.includes(includeXmiId)) addSelected(includeXmiId);
};

const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool'
    });
    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);
    }
    return;
  }
  diagramStore.selectToolItem(null);
};

const isEmpty = (): boolean => currentIds.value.length === 0;

const toggleShowAll = (): void => {
  showAll.value = !showAll.value;
};

const isSplitViewSelected = (): boolean => editorStore.isSplitView

Shown(SplitView.Positions);

const togglePositionsView = (): void => {
  editorStore.toggleSplitView(SplitView.Positions);
};

const handleSidePanelToggle = (): void => {
  if (isEditing.value) {
    toggleEdit();
  }
};

const selectItemGroup = (propValue: PropertyValue): void => {
  const selection = new PlainSelection([propValue.objectIdentifier]);
  editorStore.setSelected(selection);
  const nodesToHighlight = getNodesToHighlight([propValue]);
  editorStore.setHighlighted(new PlainHighlight(nodesToHighlight));
  if (isHighlighted(propValue)) {
    editorStore.setHighlighted(
      new PlainHighlight(
        editorStore.highlighted.filter((id: string) => id !== propValue.objectIdentifier.id)
      )
    );
  }
};

onMounted(() => eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle));
onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

<style scoped>
.button-row {
  display: flex;
  padding-bottom: var(--base-spacing-2);

  & > * {
    margin-right: var(--base-spacing-2);
  }
}

.element {
  margin-top: var(--base-spacing-2);
}

.label-row-end {
  flex-grow: 1;
  display: flex;
  justify-content: flex-end;

  & svg {
    width: 2em;
    height: 2em;
  }
}

.inline {
  width: fit-content;
}

.prop-values {
  display: grid;
  grid-template-columns: [order] 2em [handle] 3em [value] auto;
  column-gap: 1em;
  margin-top: 1.5em;
}

.prop-value-group {
  display: grid;
  grid-column: 1 / span 3;
  grid-template-columns: subgrid;
}

.handle {
  grid-column: handle;
}

.multiple {
  & .handle {
    border-right: 1px solid var(--track-item-order-group-line-color);
  }

  &:hover {
    background: var(--dropdown-item-hover-color);

    & .value {
      background: var(--dropdown-item-hover-color);
    }

    & .handle {
      border-right: 1px solid var(--track-item-order-group-line-hover);
    }
  }

  &::before {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }
  &::after {
    content: '';
    height: var(--base-spacing-1);
    grid-column: order / span 3;
  }

  & > * {
    &:nth-child(-n + 3) {
      margin-top: 0;
    }

    &:last-child {
      margin-bottom: 0;
    }
  }
}

.order {
  display: flex;
  align-items: center;
  grid-column: order;
  margin-left: 1em;
  color: var(--track-item-order-font-color);

  &.title {
    margin-left: 0;
  }
}

.value {
  grid-column: value;
  margin: var(--base-spacing-1) 0 var(--base-spacing-1) 0;
}
</style>
```

In this implementation:
- The `deleteSelected` method has been updated to call `removeTrackItemFromTrack`.
- The `removeTrackItemFromTrack` method removes the track item from the track in both the Vuex store and the GoJS diagram, without deleting it entirely.

----------------
To properly investigate how the `TrackItemProperty` values are being used and deselected, let's start by tracing the key parts of the code and understanding the flow of data and interactions. 

### Step-by-Step Analysis

1. **TrackItemProperty.vue**: We need to understand how values are passed to this component and how the deselection is handled.

2. **Highlight and Selection Handling**: Check how the highlighting and selection mechanism works in this component and across the application.

3. **Infrastructure Store Interaction**: Investigate how the store interacts with these components to manage state and update the GoJS diagram.

### TrackItemProperty.vue Analysis

1. **Values Initialization**: Identify how values are initialized and managed in the `TrackItemProperty.vue`.

2. **Deselect Functionality**: Understand how the existing deselect functionality works, including the methods and events involved.

Let's begin by focusing on the relevant parts in `TrackItemProperty.vue`. Specifically, we should review the methods and events tied to selection and deselection. This might lead us to additional files or parts of the codebase that handle the actual state management and diagram updates.

#### TrackItemProperty.vue

```vue
<template>
  <!-- Existing template structure -->
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';
import { extractXmiId } from '@ebitoolmx/eclipse-types';
import { useEditorStore } from '@/stores/editor.js';
import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useProductsStore } from '@/stores/products.js';

const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const productsStore = useProductsStore();

const props = defineProps<{
  id: string;
  value?: PropertyValue[];
  availableValues?: PropertyValue[];
  category?: string;
}>();

const deleteValue = async (propValue: PropertyValue): Promise<void> => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);

  try {
    await infrastructureStore.removeItemFromTrack({
      productId: productsStore.activeProductId,
      itemId: excludeXmiId
    });
  } catch (error) {
    console.error('Failed to remove item from track:', error);
  }

  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};

// Existing setup code...
</script>
```

### Next Steps

To better understand how these values are managed and deselected, please provide the following:

1. **Methods or Event Handlers in `TrackItemProperty.vue`**: Check if there are any methods or event handlers related to selection/deselection or value management in `TrackItemProperty.vue`.

2. **Selection and Highlighting Methods**: Any methods or services used for managing selections and highlights.

3. **State Management and Store Interaction**: Look for interactions with the store (`editorStore`, `infrastructureStore`) for state management.

### Files to Investigate

1. **Any Store Files Handling Track Items**: We need to look for files in the store (`editorStore`, `infrastructureStore`) where track items or their selections are managed.

2. **GoJS Integration Points**: Any files where GoJS diagrams are initialized or updated, especially those interacting with track items.

3. **SearchProperties.vue or Related Components**: Since `TrackItemProperty.vue` might interact with `SearchProperties.vue`, understanding this interaction might be crucial.

Please provide these details or any files you think are involved in the selection and deselection process. This will help us get a complete understanding of the flow and how to properly implement the removal of items from the track while keeping the existing functionality intact.

-----
You are correct; there isn't a method named `updateDomainObject` in the provided `infrastructure.ts` code. Instead, we need to update the track's data directly using the existing `updateData` method after modifying the track items. We'll also ensure that the changes are persisted by interacting with the backend service if necessary.

Here's the corrected approach to implement the removal of an item from a track:

### Step 1: Modify `TrackItemProperty.vue`

Ensure the `deleteValue` method calls `removeItemFromTrack`.

#### `TrackItemProperty.vue`

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { extractXmiId } from '@ebitoolmx/eclipse-types';
import { useEditorStore } from '@/stores/editor.js';
import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useProductsStore } from '@/stores/products.js';

const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const productsStore = useProductsStore();

const props = defineProps<{
  id: string;
  value?: PropertyValue[];
  availableValues?: PropertyValue[];
  category?: string;
}>();

const deleteValue = async (propValue: PropertyValue): Promise<void> => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);

  try {
    await infrastructureStore.removeItemFromTrack({
      productId: productsStore.activeProductId,
      itemId: excludeXmiId
    });
  } catch (error) {
    console.error('Failed to remove item from track:', error);
  }

  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};

// Existing setup code...
</script>
```

### Step 2: Update `infrastructure.ts`

Add a new method `removeItemFromTrack` to handle the removal of an item from a track:

#### `infrastructure.ts`

```typescript
export const useInfrastructureStore = defineStore('infrastructure', () => {
  const appStore = useAppStore();
  const productsStore = useProductsStore();
  const editorStore = useEditorStore();

  // Existing state and methods...

  const removeItemFromTrack = async ({ productId, itemId }: { productId: string; itemId: XmiId }): Promise<void> => {
    try {
      // Find the track that contains the item
      const trackContainingItem = tracks.value.find(track => track.domain.trackItems.some(item => item.reference === itemId));
      if (!trackContainingItem) {
        throw new Error(`Track containing item with id ${itemId} not found`);
      }

      // Update the track's items to exclude the item
      const updatedTrackItems = trackContainingItem.domain.trackItems.filter(item => item.reference !== itemId);
      const updatedTrack = {
        ...trackContainingItem,
        domain: {
          ...trackContainingItem.domain,
          trackItems: updatedTrackItems
        }
      };

      // Update the store's state
      updateData([updatedTrack]);

      // Call the backend service to persist the change
      await infrastructureService.submitEditRequests(productId, [
        {
          type: EditRequestType.Modify,
          xmiId: updatedTrack.xmiId,
          properties: {
            trackItems: updatedTrackItems.map(item => ({ reference: item.reference }))
          }
        }
      ]);

      // Ensure GoJS diagram updates to reflect the change
      gojsDiagram.diagram.model.startTransaction('removeItemFromTrack');
      const trackNode = gojsDiagram.diagram.findNodeForKey(trackContainingItem.xmiId);
      if (trackNode) {
        trackNode.updateRelationshipsFromData();
      }
      gojsDiagram.diagram.model.commitTransaction('removeItemFromTrack');

    } catch (err) {
      loggingService.error('Failed to remove item from track', { cause: err });
      throw err;
    }
  };

  return {
    // Existing state and methods...
    removeItemFromTrack,
  };
});
```

### Summary

1. **Update `TrackItemProperty.vue`**: Ensure the `deleteValue` method calls `removeItemFromTrack` from `infrastructure.ts`.
2. **Add `removeItemFromTrack` Method**: This method updates the track's data to exclude the item and ensures the GoJS diagram reflects this change by using `updateData` and calling the backend service with `submitEditRequests`.

By following these steps, you should be able to remove an item from a track while keeping it on the GoJS diagram, updating the state, and persisting the changes correctly. Let me know if you need further adjustments or details.


-----
Got it. You want to remove the object from the track while retaining its presence on the GoJS diagram. This involves updating the track's data to exclude the item while keeping the item's node in the diagram.

### Steps to Implement the Removal from Track Property

1. **Locate the Track Update Logic**:
   - Identify how the tracks are managed and updated within the GoJS diagram.

2. **Modify the Track's Data**:
   - When an item is removed, update the track's data to exclude the item.

3. **Ensure GoJS Diagram Reflects the Update**:
   - Ensure the GoJS diagram updates to reflect the removal of the item from the track.

### Step 1: Locate the Track Update Logic

We'll start by understanding how tracks and their items are managed. The relevant logic seems to be in `infrastructure.ts`.

### Step 2: Modify the Track's Data

We need to update the track's data to exclude the item. This will involve modifying the state and ensuring the changes are reflected in the GoJS diagram.

### Step 3: Ensure GoJS Diagram Reflects the Update

Update the GoJS diagram to reflect the changes in the track's data.

#### Update `TrackItemProperty.vue`

First, modify the `deleteValue` method to update the track's data:

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { extractXmiId } from '@ebitoolmx/eclipse-types';
import { useEditorStore } from '@/stores/editor.js';
import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useProductsStore } from '@/stores/products.js';

const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const productsStore = useProductsStore();

const props = defineProps<{
  id: string;
  value?: PropertyValue[];
  availableValues?: PropertyValue[];
  category?: string;
}>();

const deleteValue = async (propValue: PropertyValue): Promise<void> => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);

  try {
    await infrastructureStore.removeItemFromTrack({
      productId: productsStore.activeProductId,
      itemId: excludeXmiId
    });
  } catch (error) {
    console.error('Failed to remove item from track:', error);
  }

  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};

// Existing setup code...
</script>
```

#### Update `infrastructure.ts`

Add a new method `removeItemFromTrack` to handle the removal of an item from a track:

```typescript
export const useInfrastructureStore = defineStore('infrastructure', () => {
  const appStore = useAppStore();
  const productsStore = useProductsStore();
  const editorStore = useEditorStore();

  // Existing state and methods...

  const removeItemFromTrack = async ({ productId, itemId }: { productId: string; itemId: XmiId }): Promise<void> => {
    try {
      // Find the track that contains the item
      const trackContainingItem = tracks.value.find(track => track.domain.trackItems.some(item => item.reference === itemId));
      if (!trackContainingItem) {
        throw new Error(`Track containing item with id ${itemId} not found`);
      }

      // Update the track's items to exclude the item
      const updatedTrackItems = trackContainingItem.domain.trackItems.filter(item => item.reference !== itemId);
      const updatedTrack = {
        ...trackContainingItem,
        domain: {
          ...trackContainingItem.domain,
          trackItems: updatedTrackItems
        }
      };

      // Update the store's state
      updateData([updatedTrack]);

      // Call the backend service to persist the change
      await infrastructureService.updateDomainObject({
        productId,
        domainObject: updatedTrack
      });

      // Ensure GoJS diagram updates to reflect the change
      gojsDiagram.diagram.model.startTransaction('removeItemFromTrack');
      const trackNode = gojsDiagram.diagram.findNodeForKey(trackContainingItem.xmiId);
      if (trackNode) {
        trackNode.updateRelationshipsFromData();
      }
      gojsDiagram.diagram.model.commitTransaction('removeItemFromTrack');

    } catch (err) {
      loggingService.error('Failed to remove item from track', { cause: err });
      throw err;
    }
  };

  return {
    // Existing state and methods...
    removeItemFromTrack,
  };
});
```

### Summary

1. **Update `TrackItemProperty.vue`**: Ensure the `deleteValue` method calls `removeItemFromTrack` from `infrastructure.ts`.
2. **Add `removeItemFromTrack` Method**: This method updates the track's data to exclude the item and ensures the GoJS diagram reflects this change.

This approach allows for removing an item from a track while keeping it on the GoJS diagram, updating the state, and persisting the changes. Let me know if you need further adjustments or details.


----

Based on the code you provided for `editor.ts` and `infrastructure.ts`, I can see that the stores are managing the state and actions related to the editor and infrastructure. We will leverage these stores to handle the deletion of track items from both the UI and the GoJS diagram.

### Steps to Implement the Deletion

1. **Identify the Methods to Use**:
   - We'll use the `deleteDomainObject` method from `infrastructure.ts` to remove items from the GoJS diagram.
   - Ensure that `deleteValue` in `TrackItemProperty.vue` calls this method.

2. **Modify `TrackItemProperty.vue`**:
   - Update the `deleteValue` method to also call the `deleteDomainObject` method from `infrastructure.ts`.

3. **Ensure Proper Removal from GoJS Diagram**:
   - Verify that the `deleteDomainObject` method in `infrastructure.ts` handles the removal from GoJS correctly.

### Step 1: Identify the Methods to Use

The `deleteDomainObject` method in `infrastructure.ts` seems to handle the deletion logic. It calls an API to delete the domain object and also interacts with the `editorStore` to update the state.

### Step 2: Modify `TrackItemProperty.vue`

Update the `deleteValue` method in `TrackItemProperty.vue` to call `deleteDomainObject`.

#### `TrackItemProperty.vue`

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { extractXmiId } from '@ebitoolmx/eclipse-types';
import { useEditorStore } from '@/stores/editor.js';
import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useProductsStore } from '@/stores/products.js';

const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const productsStore = useProductsStore();

const props = defineProps<{
  id: string;
  value?: PropertyValue[];
  availableValues?: PropertyValue[];
  category?: string;
}>();

const deleteValue = async (propValue: PropertyValue): Promise<void> => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);

  try {
    await infrastructureStore.deleteDomainObject({
      productId: productsStore.activeProductId,
      xmiIds: [excludeXmiId]
    });
  } catch (error) {
    console.error('Failed to delete domain object:', error);
  }

  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};

const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};

// Existing setup code...
</script>
```

### Step 3: Ensure Proper Removal from GoJS Diagram

The `deleteDomainObject` method in `infrastructure.ts` is responsible for updating the state and making the API call to delete the domain object. Ensure this method is correctly interacting with the GoJS diagram.

#### `infrastructure.ts`

```typescript
export const useInfrastructureStore = defineStore('infrastructure', () => {
  // Existing state and methods...

  const deleteDomainObject = async ({
    productId,
    xmiIds
  }: DeleteDomainObjectPayload): Promise<void> => {
    try {
      editorStore.domainObjectsDeleted(xmiIds);

      // Assuming infrastructureService.deleteDomainObject handles the backend deletion
      await infrastructureService.deleteDomainObject({
        productId,
        xmiIds
      });

      // Remove nodes from the GoJS diagram
      xmiIds.forEach(xmiId => {
        const node = gojsDiagram.diagram.findNodeForKey(xmiId);
        if (node) {
          gojsDiagram.diagram.remove(node);
        }
      });

      // Update the state to reflect the deletion
      const domainObjects = xmiIds.map(xmiId => getDomainObject.value(xmiId)).filter(isNotEmpty);
      deleteData(domainObjects);
    } catch (err) {
      loggingService.error('Failed to delete domain object', { cause: err });
      toastService.displayError({
        title: 'error.requestErrorTitle',
        text: 'error.failedToDeleteDomainObject'
      });

      throw err;
    }
  };

  return {
    // Existing state and methods...
    deleteDomainObject,
  };
});
```

### Summary

1. **Update `TrackItemProperty.vue`**: Ensure the `deleteValue` method calls `deleteDomainObject` from `infrastructure.ts`.
2. **Ensure `infrastructure.ts` Handles GoJS Removal**: Verify that `deleteDomainObject` removes nodes from the GoJS diagram and updates the state accordingly.

This approach ensures that when an item is deleted from the side panel, it is also removed from the GoJS diagram, aligning with your project's existing structure and methods.

----
Understood, let's work within the existing structure of the project. We'll start by tracing how the highlight and deselect functionality is currently implemented. This will help us understand how to extend it to fully remove items from the GoJS diagram.

### Step-by-Step Process

1. **Trace Highlight/Deselect Logic**:
   - Identify where and how the highlight/deselect functionality is handled in both the GoJS diagram and the side panel.

2. **Locate the Existing Methods**:
   - Find existing methods or services that handle the removal of items from the diagram and side panel.

3. **Integrate Removal Functionality**:
   - Use the identified methods to remove items from the GoJS diagram when they are deleted from the side panel.

### Step 1: Trace Highlight/Deselect Logic

Based on your initial code, it seems the deselection is handled in `TrackItemProperty.vue`. We need to find where this logic interfaces with GoJS.

#### `TrackItemProperty.vue`

Let's examine the `deleteValue` function and trace its execution path:

```vue
const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  // Other logic...
};
```

The `deleteSelected` function is a good starting point. Let's find its definition and understand how it interacts with GoJS.

### Step 2: Locate the Existing Methods

Find the `deleteSelected` method and see how it handles deselecting items:

#### `TrackItemProperty.vue`

```vue
const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};
```

This method interacts with the `editorStore`. We'll need to look at the `editorStore` to understand how it manages selections and highlights.

#### `editorStore` (Vuex Store or Pinia Store)

Look at the methods `setHighlighted` and `setSelected` to see how they work:

```js
// Assume this is part of your editorStore

export const useEditorStore = defineStore('editor', {
  state: () => ({
    highlighted: new PlainHighlight([]),
    selected: new PlainSelection([]),
    // other states...
  }),
  actions: {
    setHighlighted(highlight) {
      this.highlighted = highlight;
      // Possibly other logic...
    },
    setSelected(selection) {
      this.selected = selection;
      // Possibly other logic...
    },
  },
});
```

### Step 3: Integrate Removal Functionality

We need to ensure that when `deleteValue` is called, it not only deselects the item but also removes it from the GoJS diagram.

#### `TrackItemProperty.vue`

Update the `deleteValue` method to include the removal of the GoJS node:

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { useEditorStore } from '@/stores/editor.js'; // Adjust the import path
import { useInfrastructureStore } from '@/stores/infrastructure.js'; // Adjust the import path

const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();

// Existing setup code...

const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  
  // Remove the item from the infrastructure store (and thus from GoJS)
  infrastructureStore.deleteDomainObject({
    productId: productsStore.activeProductId,
    xmiIds: [excludeXmiId],
  });
  
  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};
</script>
```

Ensure `deleteDomainObject` in `infrastructureStore` also handles removing the item from the GoJS diagram:

#### `infrastructureStore.js`

```js
export const useInfrastructureStore = defineStore('infrastructure', {
  state: () => ({
    // state properties...
  }),
  actions: {
    deleteDomainObject({ productId, xmiIds }) {
      // Existing logic to delete domain object...

      // Additional logic to remove from GoJS diagram
      xmiIds.forEach((xmiId) => {
        const node = gojsDiagram.diagram.findNodeForKey(xmiId);
        if (node) {
          gojsDiagram.diagram.remove(node);
        }
      });
    },
    // other actions...
  },
});
```

### Summary

1. **Trace the Logic**: Identify the `deleteSelected` method and trace how it interacts with GoJS.
2. **Locate Methods**: Understand how `editorStore` and `infrastructureStore` manage selections and highlights.
3. **Integrate Removal**: Update `deleteValue` to call the appropriate method in `infrastructureStore` that ensures the item is removed from GoJS.

By following these steps, you should be able to fully remove items from both the GoJS diagram and the side panel when the user clicks the "x" button. Let me know if you need further adjustments or details.
------
You are correct in your concerns. Typically, methods like `removeTrackItem` would be better placed in a service or a composable that handles GoJS logic, ensuring separation of concerns and better modularity. Given your project's architecture, it seems appropriate to place such logic in a dedicated service or composable.

Let's refactor the approach:

1. **Create a GoJS Service or Composable**:
   - This service/composable will handle all interactions with the GoJS diagram.

2. **Use the GoJS Service in `InfrastructureEditor.vue`**:
   - Initialize and manage the GoJS diagram using the service.

3. **Call the GoJS Service from `TrackItemProperty.vue`**:
   - Use the service to remove items from the GoJS diagram.

### Step 1: Create a GoJS Service or Composable

Let's create a composable `useGojsService.js` to manage GoJS interactions.

#### `useGojsService.js`

```javascript
import * as go from 'gojs';

let diagram;

export function useGojsService() {
  const initializeDiagram = (elementId) => {
    diagram = new go.Diagram(elementId);
    // Initialize your diagram with nodes, links, templates, etc.
  };

  const removeTrackItem = (xmiId) => {
    if (diagram) {
      const node = diagram.findNodeForKey(xmiId);
      if (node) {
        diagram.remove(node);
      }
    }
  };

  return {
    initializeDiagram,
    removeTrackItem,
  };
}
```

### Step 2: Use the GoJS Service in `InfrastructureEditor.vue`

Integrate the composable in `InfrastructureEditor.vue`:

#### `InfrastructureEditor.vue`

```vue
<template>
  <div id="infrastructure-editor">
    <app-mounted-diagram
      v-if="gojsDiagram.allTemplatesLoaded.value"
      :target="gojsDiagram.diagramTarget"
      :diagram="gojsDiagram.diagramTarget"
      @mounted="onDiagramMounted"
    />
    <template v-else>
      <app-async-app-loading-screen />
    </template>
  </div>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount } from 'vue';
import { useGojsService } from '@/composables/useGojsService'; // Adjust the import path

const { initializeDiagram } = useGojsService();

const onDiagramMounted = (target: string) => {
  initializeDiagram(target);
};

// Rest of your setup logic...
</script>
```

### Step 3: Call the GoJS Service from `TrackItemProperty.vue`

Update `TrackItemProperty.vue` to use the `removeTrackItem` method from the composable:

#### `TrackItemProperty.vue`

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { useGojsService } from '@/composables/useGojsService'; // Adjust the import path

const { removeTrackItem } = useGojsService();

// Existing setup code...

const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  
  // Remove the item from GoJS diagram
  removeTrackItem(excludeXmiId);
  
  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};
</script>
```

### Summary

1. **Create `useGojsService`**: A composable to manage GoJS interactions, including initialization and item removal.
2. **Integrate in `InfrastructureEditor.vue`**: Use the composable to initialize the GoJS diagram.
3. **Use in `TrackItemProperty.vue`**: Call the `removeTrackItem` method from the composable when an item is deleted.

This approach ensures a clean separation of concerns and modularity, making the codebase easier to maintain and extend. If you encounter any issues or need further adjustments, please let me know.

-----
Based on the provided code for `InfrastructureEditor.vue`, it seems this component does initialize and manage the GoJS diagram. Now, we can proceed to add functionality to remove track items from the GoJS diagram when they are deleted from `TrackItemProperty.vue`.

### Step-by-Step Implementation

1. **Add a Method to Remove Track Items in `InfrastructureEditor.vue`**:
   - This method will handle removing the track items from the GoJS diagram.

2. **Expose the Method for Use in Other Components**:
   - Export the method so it can be imported and used in `TrackItemProperty.vue`.

3. **Update `TrackItemProperty.vue` to Call the Remove Method**:
   - When an item is deleted from `TrackItemProperty.vue`, it should also call the method to remove the item from the GoJS diagram.

### Step 1: Add Remove Method in `InfrastructureEditor.vue`

Add a method `removeTrackItem` to handle the removal of track items:

```vue
<template>
  <div id="infrastructure-editor">
    <app-mounted-diagram
      v-if="gojsDiagram.allTemplatesLoaded.value"
      :target="gojsDiagram.diagramTarget"
      :diagram="gojsDiagram.diagramTarget"
      @mounted="onDiagramMounted"
    />
    <template v-else>
      <app-async-app-loading-screen />
    </template>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, watch } from 'vue';
import * as go from 'gojs'; // Import gojs
import {
  ChangedEvent,
  DiagramEvent,
  Part,
  Link,
  Set as GoSet,
  Diagram,
  Key,
  GraphObject
} from 'gojs';

// ... (other imports)

import AppMountedDiagram from '@/components/common/mountedDiagram/MountedDiagram.vue';

// ... (other imports and setup)

const removeTrackItem = (xmiId) => {
  const node = gojsDiagram.diagram.findNodeForKey(xmiId);
  if (node) {
    gojsDiagram.diagram.remove(node);
  }
};

// Expose the removeTrackItem method
export { removeTrackItem };

// ... (rest of the code)
</script>
```

### Step 2: Update `TrackItemProperty.vue` to Call Remove Method

Update `TrackItemProperty.vue` to import and use the `removeTrackItem` method:

```vue
<script setup lang="ts">
import { computed, ref } from 'vue';
import { removeTrackItem } from '@/components/InfrastructureEditor.vue'; // Adjust the import path

// Existing setup code...

const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  
  // Remove the item from GoJS diagram
  removeTrackItem(excludeXmiId);
  
  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};
</script>
```

### Summary

1. **Add `removeTrackItem` Method**: This method finds and removes a node from the GoJS diagram based on the `xmiId`.
2. **Export the Method**: Export `removeTrackItem` from `InfrastructureEditor.vue`.
3. **Call the Method in `TrackItemProperty.vue`**: Import and call `removeTrackItem` when an item is deleted in `TrackItemProperty.vue`.

By following these steps, deleting an item from `TrackItemProperty.vue` should also remove it from the GoJS diagram in `InfrastructureEditor.vue`. If you encounter any issues or need further adjustments, please let me know.

-----
It looks like we missed importing the `nextTick` function correctly. Let me provide a complete working example with proper imports and handling.

### Updated `SearchProperties.vue`

Ensure to import `nextTick` from Vue and handle the scroll position correctly.

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList" :key="listKey">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value"
                  :is-editing="true"
                  @delete="handleDelete(value)"
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, watch, nextTick } from 'vue'; // Correct import for nextTick
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const scrollableList = ref<HTMLElement | null>(null);
const scrollPosition = ref(0);
const listKey = ref(0);
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown.value) {
    searchDropdown.value.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = async (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);

  // Force a re-render of the list by changing the key
  listKey.value += 1;

  await nextTick();

  if (scrollableList.value) {
    scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
  }
  opened.value = true; // Ensure the dropdown remains open after deletion
};

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

### Key Points:
1. **Correct Import**: Import `nextTick` from Vue correctly.
2. **Scroll Position Handling**: 
   - Store the scroll position before deleting an item.
   - Use a key (`listKey`) to force a re-render of the list.
   - Restore the scroll position using `nextTick` after the DOM has updated.

This should correctly handle the scroll position, ensuring that the dropdown doesn't jump back to the top after an item is deleted.



-----
Sure, let's try a different approach. Instead of relying on `nextTick` to restore the scroll position, let's directly handle the DOM updates and ensure the scroll position is restored correctly.

### Different Approach: Use a Key to Force Re-render and Maintain Scroll Position

1. **Capture and Restore Scroll Position Using a Key**:
   - Capture the scroll position before the deletion.
   - Use a key to force the re-render of the list and restore the scroll position.

### Updated `SearchProperties.vue`

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList" :key="listKey">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value"
                  :is-editing="true"
                  @delete="handleDelete(value)"
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, watch } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const scrollableList = ref<HTMLElement | null>(null);
const scrollPosition = ref(0);
const listKey = ref(0);
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown.value) {
    searchDropdown.value.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);

  // Force a re-render of the list by changing the key
  listKey.value += 1;

  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
    }
    opened.value = true; // Ensure the dropdown remains open after deletion
  });
};

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

### Key Points:
1. **Capture and Restore Scroll Position Using a Key**:
   - Use a key (`listKey`) to force the re-render of the list.
   - Capture the scroll position before deletion.
   - Restore the scroll position after the DOM updates using `nextTick`.

This approach ensures the scroll position is maintained even if the list is re-rendered. By using a key to force the re-render, we can better control the behavior of the dropdown and ensure the scroll position is preserved.
-----

Let's address the issues step by step:

### 1. TypeScript Error
The TypeScript error indicates that `MinimalSearchValue` is not assignable to the expected type for the `value` property in `AppDeletableIndicator`. You need to ensure the types match. 

First, let's check if `MinimalSearchValue` can be cast to the expected type, or if you need to transform it.

### 2. Scroll Position Issue
The scroll position issue might be due to the fact that the dropdown is getting rerendered and losing the scroll position. We need to ensure the position is correctly captured and restored.

Here’s the updated solution:

### Updated `SearchProperties.vue`
Ensure the `value` passed to `AppDeletableIndicator` matches the expected type and the scroll position is maintained.

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value as any"  <!-- Cast to any or appropriate type -->
                  :is-editing="true"
                  @delete="handleDelete(value)"
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, watch, nextTick } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const scrollableList = ref<HTMLElement | null>(null);
const scrollPosition = ref(0);
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown.value) {
    searchDropdown.value.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
    }
    opened.value = true; // Ensure the dropdown remains open after deletion
  });
};

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

### Key Changes:
1. **Type Casting**: Cast `value` to `any` to resolve the TypeScript error. Ideally, you should ensure `value` matches the expected type, but casting to `any` can be a temporary workaround.
2. **Scroll Position Handling**: 
   - Store the scroll position before deletion.
   - Restore the scroll position using `nextTick` to ensure the DOM has updated.

This should address both the TypeScript error and the scroll position issue. Let me know if this resolves the problem or if further adjustments are needed.


------

Let's address the issues step by step:

### 1. TypeScript Error
The TypeScript error indicates that `MinimalSearchValue` is not assignable to the expected type for the `value` property in `AppDeletableIndicator`. You need to ensure the types match. 

First, let's check if `MinimalSearchValue` can be cast to the expected type, or if you need to transform it.

### 2. Scroll Position Issue
The scroll position issue might be due to the fact that the dropdown is getting rerendered and losing the scroll position. We need to ensure the position is correctly captured and restored.

Here’s the updated solution:

### Updated `SearchProperties.vue`
Ensure the `value` passed to `AppDeletableIndicator` matches the expected type and the scroll position is maintained.

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value as any"  <!-- Cast to any or appropriate type -->
                  :is-editing="true"
                  @delete="handleDelete(value)"
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, watch, nextTick } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const scrollableList = ref<HTMLElement | null>(null);
const scrollPosition = ref(0);
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown.value) {
    searchDropdown.value.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
    }
    opened.value = true; // Ensure the dropdown remains open after deletion
  });
};

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

### Key Changes:
1. **Type Casting**: Cast `value` to `any` to resolve the TypeScript error. Ideally, you should ensure `value` matches the expected type, but casting to `any` can be a temporary workaround.
2. **Scroll Position Handling**: 
   - Store the scroll position before deletion.
   - Restore the scroll position using `nextTick` to ensure the DOM has updated.

This should address both the TypeScript error and the scroll position issue. Let me know if this resolves the problem or if further adjustments are needed.

------
To address the issue of maintaining the scroll position in the dropdown when an item is removed, we need to ensure that the scroll position is correctly captured and restored after the dropdown is updated. Let's further refine the approach:

1. **Capture the Scroll Position Before Deletion**: Ensure the scroll position is captured before the item is deleted.
2. **Restore the Scroll Position After Deletion**: Use `nextTick` to restore the scroll position after the DOM has been updated.

Here's the refined implementation for `SearchProperties.vue`:

### `SearchProperties.vue`

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value"
                  :is-editing="true"
                  @delete="handleDelete(value)"
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, watch, nextTick } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const scrollableList = ref<HTMLElement | null>(null);
const scrollPosition = ref(0);
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown.value) {
    searchDropdown.value.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
    }
    opened.value = true; // Ensure the dropdown remains open after deletion
  });
};

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

### Explanation:

1. **Correct Import**: Ensure `AppDeletableIndicator` is correctly imported and registered.
2. **Scroll Position Handling**:
   - Store the scroll position in `scrollPosition` before deleting an item.
   - Use `nextTick` to restore the scroll position after the DOM updates.
   - Ensure the dropdown remains open by setting `opened.value` to `true` after restoring the scroll position.

This implementation should maintain the scroll position in the dropdown when an item is deleted, allowing users to remove multiple items from the same spot without needing to scroll back to their previous position.

-----
Here's the updated `SearchProperties.vue` with the correct import and refactoring to maintain the scroll position.

### `SearchProperties.vue`

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value"
                  :is-editing="true"
                  @delete="handleDelete(value)"
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, watch, nextTick } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const scrollableList = ref<HTMLElement | null>(null);
const scrollPosition = ref(0);
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown.value) {
    searchDropdown.value.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
    }
    opened.value = true; // Ensure the dropdown remains open after deletion
  });
};

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

### Explanation:
1. **Correct Import**: Ensure `AppDeletableIndicator` is correctly imported and registered.
2. **Scroll Position Handling**: 
   - Store the scroll position in `handleDelete`.
   - Use `nextTick` to wait until the DOM updates and then restore the scroll position.
   
This should maintain the scroll position in the dropdown when an item is deleted, allowing for a smooth user experience when dealing with long lists.

-----
To improve the behavior of the dropdown by maintaining the current scroll position when an item is removed, you can store the scroll position before the item is removed and then restore it after the dropdown is updated. 

Here's how you can achieve that:

1. **Add a ref for the dropdown and a data property for the scroll position in `SearchProperties.vue`**:

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable" ref="scrollableList">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value"
                  :is-editing="true"
                  @delete="handleDelete(value)" <!-- Bind the handleDelete method -->
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, VueElement, watch } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null); // Add ref for scrollable list
const scrollPosition = ref(0); // Add ref for scroll position
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  if (scrollableList.value) {
    scrollPosition.value = scrollableList.value.scrollTop; // Store the scroll position
  }
  emit('delete', value);
  setTimeout(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollPosition.value; // Restore the scroll position
    }
    opened.value = true; // Ensure the dropdown remains open after deletion
  }, 0);
};

watch(searchString, value => emit('update:modelValue', value));
</script>

<style scoped>
/* Existing styles */
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

In this solution, we:

- Added a `ref` for the scrollable list (`scrollableList`).
- Added a `ref` to store the current scroll position (`scrollPosition`).
- Modified the `handleDelete` method to store the scroll position before deleting an item and then restore the scroll position after the deletion.

This will ensure that the dropdown maintains its scroll position when an item is removed, allowing users to remove multiple items from the same spot without needing to scroll back to their previous position.




--------

Let's correct this by ensuring that the `handleDelete` method in `SearchProperties.vue` is correctly bound to the delete event, and we'll make sure the dropdown stays open.

### Step-by-Step Solution

1. **Update `SearchProperties.vue` to correctly handle the delete event:**

```vue
<template>
  <div class="search-property-wrapper">
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
      @click="opened = true"
      @blur="closeDropdown"
    >
      <app-icon-button
        v-if="searchString"
        name="times"
        class="clickable"
        stop-propagation
        @click="clearSearch"
      />
    </app-input>
    <div
      v-if="opened"
      ref="searchDropdown"
      tabindex="0"
      class="results"
      @blur="closeDropdown"
      @mousedown="focusDropdown"
    >
      <ol class="scrollable">
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
              <slot name="selectedValue" v-bind="value">
                <app-deletable-indicator
                  :value="value"
                  :is-editing="true"
                  @delete="handleDelete(value)" <!-- Bind the handleDelete method -->
                >{{ parseSearchValue(value) }}</app-deletable-indicator>
              </slot>
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
import { computed, ref, VueElement, watch } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
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
  delete: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
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
  opened.value = false;
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = () => {
  if (document.activeElement === searchDropdown.value) return;
  opened.value = false;
  emit('blur', searchString.value);
};

const handleDelete = (value: T) => {
  emit('delete', value);
  setTimeout(() => {
    opened.value = true; // Ensure the dropdown remains open after deletion
  }, 0);
};

watch(searchString, value => emit('update:modelValue', value));
</script>

<style scoped>
/* Existing styles */
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

2. **Ensure `TrackItemProperty.vue` correctly utilizes the delete event:**

Update `TrackItemProperty.vue` to ensure it correctly handles the delete event without closing the dropdown.

```vue
<template>
  <app-property-container
    :id="id"
    v-bind="$attrs"
    :full-width="true"
    :editing="isEditing"
    :editable="editorStore.isEditMode"
    icon="picker"
    :title="t('common.edit')"
    :value-size="value.length || 0"
    property-type="trackItemsReference"
    @toggle-editing="toggleEdit"
  >
    <template #label-row-end>
      <div class="label-row-end">
        <app-icon-button
          :name="showAll ? 'chevron-up' : 'chevron-down'"
          :title="t('common.all')"
          @click="toggleShowAll"
        />
      </div>
    </template>
    <div class="element">
      <div class="button-row">
        <app-indicator
          v-if="!isEmpty()"
          class="inline"
          :highlighted="isAllValuesHighlighted()"
          :highlightable="true"
          :type="IndicatorType.Main"
          @click="highlight(value)"
        >
          {{ t('common.all') }}
        </app-indicator>
        <app-indicator
          class="inline"
          :highlightable="true"
          :highlighted="isSplitViewSelected()"
          @click="togglePositionsView()"
        >
          {{ t('common.positions') }}
        </app-indicator>
      </div>
      <div v-if="showAll">
        <app-search-properties
          :available-values="availableValues"
          :selected-values="value"
          :enabled="!$attrs.readonly"
          :input-placeholder="t('properties.search')"
          :label-key="'properties.labels.' + id"
          @selected="addValue"
          @delete="deleteValue" <!-- Ensure this line exists to handle delete events -->
          ><template #selectedValue="selectedValue">
            <app-deletable-indicator
              :value="selectedValue"
              :is-editing="isEditing"
              @delete="deleteValue(selectedValue)"
              @select="selectItemGroup(selectedValue)"
              @highlight="highlight([selectedValue])" /></template
        ></app-search-properties>
        <div class="prop-values">
          <div class="prop-value-group">
            <div class="order title">{{ t('properties.order') }}</div>
            <div class="value">{{ t('properties.trackItems') }}</div>
          </div>
          <div
            v-for="(group, index) in groupedTrackItems"
            :key="index"
            :class="{ multiple: group.length > 1 }"
            class="prop-value-group"
          >
            <template v-for="[propValue, i] in group" :key="propValue.displayText">
              <div class="order">{{ i }}</div>
              <div class="handle" />
              <app-deletable-indicator
                class="value"
                :value="propValue"
                :is-editing="isEditing"
                @delete="deleteValue(propValue)"
                @select="selectItemGroup(propValue)"
                @highlight="highlight([propValue])"
              />
            </template>
          </div>
        </div>
      </div>
    </div>
  </app-property-container>
</template>
<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useI18n } from 'vue-i18n';
import { PropertyValue } from '@ebitoolmx/cbss-types';
import { XmiId } from '@ebitoolmx/eclipse-types';
import { extractXmiId } from '@ebitoolmx/ebitool-classic-types';
import { isNotEmpty } from '@ebitoolmx/predicates';
import { ToolType, ToolItemCategory } from '@/typings/tools.js';
import { SplitView } from '@/typings/splitView.js';
import { IndicatorType } from '@/typings/indicator.js';
import { PlainSelection } from '@/typings/selection/PlainSelection.js';
import { PlainHighlight } from '@/typings/highlight/PlainHighlight.js';
import AppDeletableIndicator from '@/components/common/indicator/DeletableIndicator.vue';
import AppIndicator from '@/components/common/indicator/Indicator.vue';
import AppPropertyContainer from '@/components/common/sidePanelElements/PropertyContainer.vue';
import AppSearchProperties from '@/components/common/search/SearchProperties.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { eventService, EventType } from '@/services/event.js';
import { useInfrastructureStore } from '@/stores/infrastructure.js';
import { useDiagramStore } from '@/stores/diagram.js';
import { useEditorStore } from '@/stores/editor.js';
import { isMXObjectIdentifier } from '@/typings/selection';

defineOptions({ name: 'TrackItemProperty' });

const props = withDefaults(
  defineProps<{
    id: string;
    value?: PropertyValue[];
    availableValues?: PropertyValue[];
    category?: string;
  }>(),
  {
    value: () => [],
    availableValues: () => [],
    category: 'Unknown'
  }
);
const emit = defineEmits<{ change: [xmiIds: XmiId[]] }>();
const { t } = useI18n();
const editorStore = useEditorStore();
const infrastructureStore = useInfrastructureStore();
const diagramStore = useDiagramStore();
const showAll = ref<boolean>(false);
const isEditing = computed<boolean>(
  () =>
    diagramStore.selectedToolItem?.category === props.category &&
    diagramStore.selectedToolItem.toolType === ToolType.PickerTool
);
const groupedTrackItems = computed<Array<Array<[PropertyValue, number]>>(() => {
  const groupedByMileage = props.value.reduce(
    (acc: Record<string, Array<[PropertyValue, number]>>, item: PropertyValue, i: number) => {
      const domainObject = infrastructureStore.getTrackItem(item.objectIdentifier.id);
      const mileage = domainObject?.domain.position?.mileage;
      if (mileage && !acc[mileage]) {
        acc[mileage] = [];
      }
      if (mileage && acc[mileage]) {
        acc[mileage].push([item, i + 1]);
      }
      return acc;
    },
    {}
  );
  const formattedArray = Object.values(groupedByMileage);
  return formattedArray;
});
const currentIds = computed<string[]>(() => props.value.map(p => extractXmiId(p.object.reference)));
function getNodesToHighlight(propValues: PropertyValue[]): string[] {
  const nodes = propValues.map(value => value.objectIdentifier.id);
  return nodes.filter(isNotEmpty);
}
function isAllValuesHighlighted(): boolean {
  if (editorStore.highlighted.isEmpty()) return false;
  const nodesToHighlight = getNodesToHighlight(props.value).filter(Boolean);
  // Returns true if all or more values are highlighted.
  // selecting an area group should indicate that all items are highlighted.
  // Even though the items belonging to the areas inside the group isn’t directly connected to the area group
  return nodesToHighlight.every(node => editorStore.highlighted.expandedIds.includes(node));
}
function isHighlighted(id: PropertyValue): boolean {
  const [nodeToHighlight] = getNodesToHighlight([id]);
  return editorStore.highlighted.includes(nodeToHighlight);
}
const highlight = (propValues: PropertyValue[]): void => {
  const nodesToHighlight = getNodesToHighlight(propValues);
  editorStore.toggleHighlighted(new PlainHighlight(nodesToHighlight));
  eventService.emit(EventType.CenterHighlighted);
  for (const propValue of propValues) {
    const layer = propValue.object.eClass;
    diagramStore.setLayerVisibility([{ layer, visible: true }]);
  }
};
const deleteSelected = (xmiId: XmiId): void => {
  emit(
    'change',
    currentIds.value.filter(id => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
  );
  const newSelection = editorStore.selected.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};
const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(new PlainHighlight([...editorStore.highlighted.expandedIds, xmiId]));
  emit('change', [...currentIds.value, xmiId]);
};
const addValue = (propValue: PropertyValue): void => {
  const includeXmiId = extractXmiId(propValue.object.reference);
  if (!currentIds.value.includes(includeXmiId)) addSelected(includeXmiId);
};
const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  setTimeout(() => {
    showAll.value = true; // Ensure dropdown remains open
  }, 0);
};
const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool'
    });
    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);
    }
    return;
  }
  diagramStore.selectToolItem(null);
};
const isEmpty = (): boolean => currentIds.value.length === 0;
const toggleShowAll = (): void => {
  showAll.value = !showAll.value;
};
const isSplitViewSelected = (): boolean => editorStore.isSplitViewShown(SplitView.Positions);
const togglePositionsView = (): void => {
  editorStore.toggleSplitView(SplitView.Positions);
};
const handleSidePanelToggle = (): void => {
  if (isEditing.value) {
    toggleEdit();
  }
};
const selectItemGroup = (propValue: PropertyValue): void => {
  const selection = new PlainSelection([propValue.objectIdentifier]);
  editorStore.setSelected(selection);
  const nodesToHighlight = getNodesToHighlight([propValue]);
  editorStore.setHighlighted(new PlainHighlight(nodesToHighlight));
  if (isHighlighted(propValue)) {
    editorStore.setHighlighted(
      new PlainHighlight(
        editorStore.highlighted.filter((id: string) => id !== propValue.objectIdentifier.id)
      )
    );
  }
};
onMounted(() => eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle));
onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

<style scoped>
.button-row {
  display: flex;
  padding-bottom: var(--base-spacing-2);
  & > * {
    margin-right: var(--base-spacing-2);
  }
}
.element {
  margin-top: var(--base-spacing-2);
}
.label-row-end {
  flex-grow: 1;
  display: flex;
  justify-content: flex-end;
  & svg {
    width: 2em;
    height: 2em;
  }
}
.inline {
  width: fit-content;
}
.prop-values {
  display: grid;
  grid-template-columns: [order] 2em [handle] 3em [value] auto;
  column-gap: 1em;
  margin-top: 1.5em;
}
.prop-value-group {
  display: grid;
  grid-column: 1 / span 3;
  grid-template-columns: subgrid;
}
.handle {
  grid-column: handle;
}
.multiple {
  & .handle {
    border-right: 1px solid var(--track-item-order-group-line-color);
  }
  &:hover {
    background: var(--dropdown-item-hover-color);
    & .value {
      background: var(--dropdown-item-hover-color);
    }
    & .handle {
      border-right: 1px solid var(--track-item-order-group-line-hover);
    }
  }
  /* Used to make sure borders aren't extending to the edge of the groups */
  &::before {
    content:=With these updates, the `handleDelete` method should correctly handle the deletion of items and ensure the dropdown remains open. The main change is ensuring that the `@delete` event in `SearchProperties.vue` correctly calls `handleDelete` and that `setTimeout` is used to ensure the dropdown stays open after the deletion event.

Let me know if this resolves the issue or if further adjustments are needed!
