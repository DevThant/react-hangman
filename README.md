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
