To implement the required functionalities in the `SearchProperties.vue` component, you need to handle both the click outside and the escape key to close the dropdown, while ensuring the dropdown remains open for interactions within it.

Here’s the updated code with the necessary changes:

### Updated `SearchProperties.vue`

```vue
<template>
  <div class="search-property-wrapper" @click.outside="closeDropdown">
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
      @click="openDropdown"
      @keydown.esc="closeDropdown"
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
import { computed, ref, VueElement, watch } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
import {
  matchedResultsFilter,
  MinimalSearchValue,
  parseSearchValue
} from '@/components/common/search/helpers/match.js';
import { onClickOutside, useEventListener } from '@vueuse/core'; // Import VueUse hooks

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

onClickOutside(searchDropdown, closeDropdown); // Close dropdown on outside click

useEventListener('keydown', (event) => {
  if (event.key === 'Escape') {
    closeDropdown();
  }
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

### Explanation:
1. **`@click.outside` Directive**:
   - Uses `onClickOutside` from VueUse to close the dropdown when clicking outside of it.
2. **Escape Key Handling**:
   - Uses `useEventListener` from VueUse to listen for the Escape key and close the dropdown when it's pressed.
3. **Refactored Methods**:
   - `openDropdown` method to handle opening the dropdown.
   - Updated the `closeDropdown` method to remove the `event` parameter since it's no longer used.

These changes should ensure the dropdown behaves as expected, closing when clicking outside or pressing Escape, and staying open when interacting inside.
-----
It looks like the event listeners might be interfering with the opening of the dropdown. To fix this, we need to ensure that the `focusDropdown` and `closeDropdown` methods are correctly interacting with the event listeners. Here’s a revised approach:

1. Use Vue's `v-click-outside` directive to handle outside clicks.
2. Handle the `Escape` key press within the component's keydown event.

Here’s the updated code:

```vue
<template>
  <div class="search-property-wrapper" @keydown.esc="closeDropdown" tabindex="0">
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
      @click="openDropdown"
      @blur="onInputBlur"
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
      v-click-outside="closeDropdown"
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
import { computed, ref, onMounted, onBeforeUnmount, watch, VueElement } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
import {
  matchedResultsFilter,
  MinimalSearchValue,
  parseSearchValue
} from '@/components/common/search/helpers/match.js';
import vClickOutside from 'v-click-outside';

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
};

const openDropdown = () => {
  opened.value = true;
};

const closeDropdown = () => {
  opened.value = false;
  emit('blur', searchString.value);
};

const onInputBlur = () => {
  // We use this method to handle blur on the input
  setTimeout(() => {
    if (!opened.value) {
      closeDropdown();
    }
  }, 200); // Give time for click event to propagate
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

### Explanation of Changes
1. **Vue Directive for Click Outside**: Using `v-click-outside` directive to detect clicks outside the dropdown and close it.
2. **Handling ESC Key**: Added a `keydown` event listener on the `search-property-wrapper` to close the dropdown when the ESC key is pressed.
3. **Open Dropdown**: Created an `openDropdown` method to handle opening the dropdown.
4. **Input Blur Handling**: Updated `onInputBlur` method to delay the close action slightly to allow for other events to propagate.

This should ensure that the dropdown opens and closes correctly, including closing when clicking outside or pressing the ESC key.
-------
To address the issue where the dropdown doesn't close when clicking outside of it or pressing the ESC key, you need to add event listeners for clicks outside the dropdown and the ESC key. Here’s how you can implement this:

1. Add event listeners for `click` and `keydown` to handle closing the dropdown.
2. Ensure these listeners are properly removed when the component is destroyed.

Here's the updated code:

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
import { computed, ref, onMounted, onBeforeUnmount, watch, VueElement } from 'vue';
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
  remove: [value: T];
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
  opened.value = false;
  emit('blur', searchString.value);
};

const handleClickOutside = (event: MouseEvent) => {
  if (searchDropdown.value && !searchDropdown.value.contains(event.target as Node)) {
    closeDropdown();
  }
};

const handleKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Escape') {
    closeDropdown();
  }
};

onMounted(() => {
  document.addEventListener('click', handleClickOutside);
  document.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
  document.removeEventListener('click', handleClickOutside);
  document.removeEventListener('keydown', handleKeydown);
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

### Explanation of Changes
1. **Event Listeners for Outside Click**: Added `handleClickOutside` method to detect clicks outside the dropdown and close it if clicked outside.
2. **Event Listener for ESC Key**: Added `handleKeydown` method to close the dropdown when the ESC key is pressed.
3. **Lifecycle Hooks**: Added `onMounted` to set up the event listeners when the component is mounted and `onBeforeUnmount` to remove the event listeners when the component is destroyed. 

These changes ensure that the dropdown will close when clicking outside of it or pressing the ESC key, improving the UX.
-------
You're right; I mistakenly declared `filteredAvailableValues` without using it properly. Let's correct that and ensure we maintain the scroll position correctly.

Here is the corrected version of your component, with the proper use of `filteredAvailableValues` and maintaining the scroll position effectively:

### Updated Component

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
            filteredAvailableValues.length ||
            filteredSelectedValues.length
          "
        >
          <template
            v-if="filteredAvailableValues.length && enabled"
          >
            <li class="small">{{ availableLabel }}:</li>
            <li
              v-for="value of filteredAvailableValues"
              :key="parseSearchValue(value)"
              :title="parseSearchValue(value)"
              class="selectable"
              @click="select(value)"
            >
              <slot name="availableValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
            </li>
          </template>
          <template v-if="filteredSelectedValues.length">
            <li class="small">{{ selectedLabel }}:</li>
            <li
              v-for="value of filteredSelectedValues"
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
import { computed, ref, VueElement, watch, nextTick } from 'vue';
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
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
const opened = ref(false);

const localAvailableValues = ref([...props.availableValues]);
const localSelectedValues = ref([...props.selectedValues]);

const filteredAvailableValues = computed(() => matchedResultsFilter(searchString.value, localAvailableValues.value));
const filteredSelectedValues = computed(() => matchedResultsFilter(searchString.value, localSelectedValues.value));

const noMatchFound = computed<boolean>(
  () =>
    !!searchString.value &&
    !filteredSelectedValues.value.length &&
    (!props.enabled || !filteredAvailableValues.value.length)
);

const parsedLabelKey = computed<string>(() => (props.labelKey ? ` ${t(props.labelKey)}` : ''));
const availableLabel = computed<string>(() => t('properties.available') + parsedLabelKey.value);
const selectedLabel = computed<string>(() => t('properties.selected') + parsedLabelKey.value);

const clearSearch = () => {
  searchString.value = '';
  emit('clear');
};

const select = (value: T) => {
  const scrollTop = scrollableList.value?.scrollTop || 0;
  searchString.value = parseSearchValue(value);
  localSelectedValues.value.push(value);
  localAvailableValues.value = localAvailableValues.value.filter(v => parseSearchValue(v) !== parseSearchValue(value));
  emit('selected', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollTop;
    }
  });
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
};

watch(searchString, value => emit('update:modelValue', value));

// Sync local state with props
watch(
  () => props.selectedValues,
  (newValues) => {
    localSelectedValues.value = [...newValues];
  }
);

watch(
  () => props.availableValues,
  (newValues) => {
    localAvailableValues.value = [...newValues];
  }
);

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

### Explanation of Changes
1. **Filtered Available Values**: Properly declared and used `filteredAvailableValues` to ensure it is being computed and used in the template.
2. **Filtered Selected Values**: Similarly declared and used `filteredSelectedValues`.
3. **Scroll Position Handling**: Updated the `select` function to store and restore the scroll position around the item selection logic.

This version ensures that the list is correctly filtered and that the scroll position is maintained, improving the overall user experience.
----
Let's try a different approach to maintain the scroll position without refreshing the entire dropdown. We'll manually update the list and manage the state locally to avoid reactivity issues.

Here's an updated version that focuses on managing the scroll position more effectively:

### Updated Component

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
            matchedResultsFilter(searchString, localAvailableValues).length ||
            matchedResultsFilter(searchString, localSelectedValues).length
          "
        >
          <template
            v-if="matchedResultsFilter(searchString, localAvailableValues).length && enabled"
          >
            <li class="small">{{ availableLabel }}:</li>
            <li
              v-for="value of matchedResultsFilter(searchString, localAvailableValues)"
              :key="parseSearchValue(value)"
              :title="parseSearchValue(value)"
              class="selectable"
              @click="select(value)"
            >
              <slot name="availableValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
            </li>
          </template>
          <template v-if="matchedResultsFilter(searchString, localSelectedValues).length">
            <li class="small">{{ selectedLabel }}:</li>
            <li
              v-for="value of matchedResultsFilter(searchString, localSelectedValues)"
              :key="parseSearchValue(value)"
              class="selectable"
              :title="parseSearchValue(value)"
            >
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <!-- Assuming each selected item has its own delete button -->
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
import { computed, ref, VueElement, watch, nextTick } from 'vue';
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
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
const opened = ref(false);

const localAvailableValues = ref([...props.availableValues]);
const localSelectedValues = ref([...props.selectedValues]);

const filteredAvailableValues = computed<T[]>(() => {
  const valueMap = localSelectedValues.value.map(val => parseSearchValue(val));
  return localAvailableValues.value.filter(value => !valueMap.includes(parseSearchValue(value)));
});

const noMatchFound = computed<boolean>(
  () =>
    !!searchString.value &&
    !matchedResultsFilter(searchString.value, localSelectedValues.value).length &&
    (!props.enabled || !matchedResultsFilter(searchString.value, localAvailableValues.value).length)
);

const parsedLabelKey = computed<string>(() => (props.labelKey ? ` ${t(props.labelKey)}` : ''));
const availableLabel = computed<string>(() => t('properties.available') + parsedLabelKey.value);
const selectedLabel = computed<string>(() => t('properties.selected') + parsedLabelKey.value);

const clearSearch = () => {
  searchString.value = '';
  emit('clear');
};

const select = (value: T) => {
  const scrollTop = scrollableList.value?.scrollTop || 0;
  searchString.value = parseSearchValue(value);
  localSelectedValues.value.push(value);
  localAvailableValues.value = localAvailableValues.value.filter(v => parseSearchValue(v) !== parseSearchValue(value));
  emit('selected', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollTop;
    }
  });
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
};

watch(searchString, value => emit('update:modelValue', value));

// Sync local state with props
watch(
  () => props.selectedValues,
  (newValues) => {
    localSelectedValues.value = [...newValues];
  }
);

watch(
  () => props.availableValues,
  (newValues) => {
    localAvailableValues.value = [...newValues];
  }
);

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

### Explanation of Changes
1. **Local State Management**: Manage `localAvailableValues` and `localSelectedValues` locally and synchronize with the props using `watch`.
2. **Scroll Position Handling**: Store the scroll position before updating the list and restore it using `nextTick` after the update.

This should help in maintaining the scroll position and provide a smoother user experience.

-----

It is possible to maintain the scroll position of a dropdown while updating its content. To achieve this, you need to ensure that the reactivity updates do not cause a full re-render. Instead, we should focus on efficient DOM manipulation.

Here is a more refined approach:

1. **Store Scroll Position**: Before updating the list, store the current scroll position.
2. **Update List**: Perform the necessary updates to the list (e.g., removing an item).
3. **Restore Scroll Position**: Restore the scroll position to where it was before the update.

### Updated Component

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
import { computed, ref, VueElement, watch, nextTick } from 'vue';
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
  remove: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
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

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
};

watch(searchString, value => emit('update:modelValue', value));

// Watch for updates to the selected values and maintain scroll position
watch(
  () => props.selectedValues,
  (newValues, oldValues) => {
    if (scrollableList.value) {
      const scrollTop = scrollableList.value.scrollTop;
      nextTick(() => {
        scrollableList.value.scrollTop = scrollTop;
      });
    }
  }
);

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

### Explanation of Changes
1. **State Management**: Directly use `props.selectedValues` and `props.availableValues` to ensure minimal state management.
2. **Scroll Position Handling**: Use a `watch` function to monitor changes to `props.selectedValues`. Before the update, store the current scroll position, and after the update, restore it using `nextTick`.

This approach ensures that the scroll position is maintained even when the list updates, providing a smoother user experience.


----
Understood. Let's refine the approach further by ensuring that the dropdown remains in its current state without fully re-rendering. We will focus on managing the reactivity and state changes efficiently.

Here is an updated version of the component without the redundant `remove` function and `app-icon-button`. We'll ensure that only the necessary parts of the dropdown are updated when changes occur.

### Updated Component

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
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <!-- Assuming each selected item has its own delete button -->
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
import { computed, ref, VueElement, watch, nextTick } from 'vue';
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
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
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

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
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

### Explanation
1. **State Management**: Instead of managing a separate state for `availableValues` and `selectedValues`, we directly use the props to ensure that the dropdown content changes are minimal and localized.
2. **Avoid Full Re-render**: By managing the dropdown state efficiently, we avoid unnecessary full component re-renders. The `select` method now only updates the `searchString` and emits the `selected` event without closing the dropdown.
3. **Focus Management**: Ensures that the dropdown remains focused when needed, preventing it from closing unexpectedly.

These changes should ensure that the dropdown behaves as expected, remaining open and not scrolling back to the top when items are interacted with.

----
Let's refine the approach to ensure the scroll position is correctly maintained. We will use the `watch` function to monitor changes and set the scroll position accordingly. Here's an updated version of the component:

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
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <app-icon-button name="times" class="clickable" @click.stop="remove(value)" />
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
import { computed, ref, VueElement, watch, nextTick } from 'vue';
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
  remove: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
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

const remove = (value: T) => {
  const scrollTop = scrollableList.value?.scrollTop || 0;
  emit('remove', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollTop;
    }
  });
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
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

### Explanation of Changes
1. **Scroll Position Handling**: In the `remove` method, the current scroll position is stored before emitting the `remove` event. After the event, the scroll position is restored using `nextTick`.
2. **Ref for Scrollable List**: Added a `ref` for the scrollable list to manipulate its scroll position.
3. **Ensure Scroll Restoration**: The `nextTick` function ensures that the DOM updates before restoring the scroll position.

This should help maintain the scroll position of the dropdown after an item is removed, improving the user experience.

-----
Let's refine the approach to ensure the scroll position is correctly maintained. We will use the `watch` function to monitor changes and set the scroll position accordingly. Here's an updated version of the component:

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
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <app-icon-button name="times" class="clickable" @click.stop="remove(value)" />
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
import { computed, ref, VueElement, watch, nextTick } from 'vue';
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
  remove: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
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

const remove = (value: T) => {
  const scrollTop = scrollableList.value?.scrollTop || 0;
  emit('remove', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollTop;
    }
  });
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
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

### Explanation of Changes
1. **Scroll Position Handling**: In the `remove` method, the current scroll position is stored before emitting the `remove` event. After the event, the scroll position is restored using `nextTick`.
2. **Ref for Scrollable List**: Added a `ref` for the scrollable list to manipulate its scroll position.
3. **Ensure Scroll Restoration**: The `nextTick` function ensures that the DOM updates before restoring the scroll position.

This should help maintain the scroll position of the dropdown after an item is removed, improving the user experience.

------
To improve the usability of your dropdown so that it maintains its scroll position after removing an item, you can store the scroll position before making any changes and then restore it afterward. Here's how you can achieve this:

1. Store the scroll position of the dropdown before removing an item.
2. Restore the scroll position after updating the items.

Here's the updated code to implement this:

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
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <app-icon-button name="times" class="clickable" @click.stop="remove(value)" />
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
  remove: [value: T];
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | InstanceType<typeof VueElement>>();
const scrollableList = ref<HTMLElement | null>(null);
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

const remove = (value: T) => {
  const scrollTop = scrollableList.value?.scrollTop || 0;
  emit('remove', value);
  nextTick(() => {
    if (scrollableList.value) {
      scrollableList.value.scrollTop = scrollTop;
    }
  });
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
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

### Explanation of Changes
1. **Scroll Position Handling**: In the `remove` method, the current scroll position is stored before emitting the `remove` event. After the event, the scroll position is restored using `nextTick`.
2. **Ref for Scrollable List**: Added a `ref` for the scrollable list to manipulate its scroll position.

These changes should maintain the scroll position of the dropdown even after removing items, improving the usability as desired.
------
To resolve the issue of having two 'x' buttons beside the highlighted items, you need to ensure there's only one 'x' button per item and remove the unnecessary one. 

Here is the corrected code:

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
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <app-icon-button
                name="times"
                class="clickable"
                @click.stop="remove(value)"
              />
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
  remove: [value: T];
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
};

const remove = (value: T) => {
  emit('remove', value);
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
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

### Explanation of Changes
1. **Removed Extra 'x' Button**: Ensured that the `app-icon-button` with the 'x' icon is placed only within the `selectedValue` slot. This guarantees that there is only one 'x' button per selected item.
2. **CSS Changes**: No changes needed in CSS for this specific issue as it is related to the structure of the HTML within the template. 

By ensuring the `app-icon-button` is only within the slot and not duplicated elsewhere in the template, the extra 'x' button is removed.
-----
To achieve the desired behavior where the dropdown remains open after selecting an item, but closes only when clicking outside or on the 'x' to remove an item, you can make a few modifications to the script. Here's how you can do it:

1. Modify the `select` method to not close the dropdown.
2. Add a `remove` method to handle removing items without closing the dropdown.
3. Adjust the `closeDropdown` method to not close the dropdown if the focus is still within the component.

Here is the updated code:

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
              <slot name="selectedValue" v-bind="value">{{ parseSearchValue(value) }}</slot>
              <app-icon-button
                name="times"
                class="clickable"
                @click.stop="remove(value)"
              />
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
  remove: [value: T];
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
};

const remove = (value: T) => {
  emit('remove', value);
};

const focusDropdown = () => {
  if (searchDropdown?.value?.focus) {
    searchDropdown.value.focus();
  } else if ((searchDropdown?.value as any)?.$el?.focus) {
    (searchDropdown.value as any).$el.focus();
  }
  opened.value = true;
};

const closeDropdown = (event: FocusEvent) => {
  if (searchDropdown?.value?.contains(event.relatedTarget as Node)) return;
  opened.value = false;
  emit('blur', searchString.value);
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

### Explanation of Changes
1. **Select Method**: The `opened.value = false;` line is removed from the `select` method to keep the dropdown open after selecting an item.
2. **Remove Method**: A new `remove` method is added to handle the removal of items without closing the dropdown.
3. **Close Dropdown**: The `closeDropdown` method is updated to ensure it doesn't close the dropdown if the focus is still within the component.
4. **Template Changes**: Added an `app-icon-button` inside the `selectedValue` slot for the removal of items.

This should maintain the dropdown open when selecting an item and only close it when clicking outside or on the 'x' to remove an item.
-----
Based on your current setup, it appears there might be a conflict or an incorrect logic placement that's preventing the dropdown from opening properly. Let's refactor the event listeners and the interaction logic to ensure that the dropdown opens and closes as expected.

### Issue Analysis and Correction:

1. **Redundant Event Listeners**: The `onMounted` and `onBeforeUnmount` lifecycle hooks were added twice, which might cause unexpected behavior.

2. **Click Event Handling**: The current event handling might be too aggressive in closing the dropdown, especially considering that clicks on the dropdown itself should not close it but are possibly interpreted as outside clicks.

Let's refactor the code with a clearer separation of concerns:

```vue
<script setup lang="ts" generic="T extends MinimalSearchValue">
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue';
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
}>();

const { t } = useI18n();
const searchString = ref(props.modelValue);
const searchDropdown = ref<HTMLElement | null>(null);
const opened = ref(false);

const clearSearch = () => {
  searchString.value = '';
  emit('clear');
  opened.value = false; // Close dropdown when search is cleared
};

const select = (value: T) => {
  searchString.value = parseSearchValue(value);
  emit('selected', value);
  opened.value = false; // Close dropdown when an item is selected
};

const focusDropdown = () => {
  opened.value = true;
};

const checkClickOutside = (event: MouseEvent) => {
  nextTick(() => {
    if (
      searchDropdown.value &&
      !searchDropdown.value.contains(event.target as Node) &&
      opened.value
    ) {
      closeDropdown();
    }
  });
};

const closeDropdown = () => {
  opened.value = false;
};

onMounted(() => {
  document.addEventListener('click', checkClickOutside);
  window.addEventListener('keydown', (event: KeyboardEvent) => {
    if (event.key === 'Escape') {
      closeDropdown();
    }
  });
});

onBeforeUnmount(() => {
  document.removeEventListener('click', checkClickOutside);
  window.removeEventListener('keydown', closeDropdown);
});
</script>
```

### Key Adjustments:

- **Event Handling Refactor**: Simplified and ensured no redundant event listeners.
- **Immediate Event Handling**: Using `nextTick` to delay the execution of the check for clicks outside the dropdown, ensuring that the dropdown setup completes before the event check interferes.
- **Dropdown Logic**: Clearly define when the dropdown should remain open and when it should close based on user interactions.

This setup should better handle your requirements of keeping the dropdown open when interacting with it and only closing on outside clicks or ESC key presses.
-----
The TypeScript error you're seeing is because the event parameter doesn't have an explicit type declared. TypeScript requires explicit types for a more rigorous type checking process, ensuring you are using the expected properties and methods associated with that type.

To resolve this issue, you should specify the type of the `event` parameter using the `MouseEvent` type for click events. This will inform TypeScript that `event` is a mouse event, giving you access to the appropriate properties and methods. Here's the corrected version:

```javascript
<script setup lang="ts" generic="T extends MinimalSearchValue">
import { ref, onMounted, onBeforeUnmount } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
import {
  matchedResultsFilter,
  MinimalSearchValue,
  parseSearchValue
} from '@/components/common/search/helpers/match.js';

defineOptions({ name: 'SearchProperties' });

// Your existing props, emit, and other constants setup here ...

const opened = ref(false);
const searchDropdown = ref<HTMLElement | null>(null);

const focusDropdown = () => {
  opened.value = true;
};

const closeDropdown = () => {
  opened.value = false;
};

const checkClickOutside = (event: MouseEvent) => {
  if (searchDropdown.value && !searchDropdown.value.contains(event.target as Node) && opened.value) {
    closeDropdown();
  }
};

onMounted(() => {
  document.addEventListener('click', checkClickOutside);
  window.addEventListener('keydown', (event: KeyboardEvent) => {
    if (event.key === "Escape") {
      closeDropdown();
    }
  });
});

onBeforeUnmount(() => {
  document.removeEventListener('click', checkClickOutside);
  window.removeEventListener('keydown', closeDropdown);
});

// Remaining part of your script...
</script>
```

### Changes made:

1. **Event Typing**: Added type `MouseEvent` to the `checkClickOutside` function parameter and `KeyboardEvent` to the keydown event listener.
2. **Consistent Type Checking**: By using explicit typing, TypeScript can now ensure the use of properties and methods is valid for the specified event types.

This should address the TypeScript error and enable proper event handling in your Vue component setup.


-----
You're right about the issues with the `focusEvent` function. It's not being used, and the typo with `.closet` instead of `.closest` would indeed cause an error.

Let's correct these issues by integrating the correct event handling directly into the component lifecycle without the unused `focusEvent` function. Here's a revised approach focusing directly on handling clicks and keydown events properly:

```javascript
<script setup lang="ts" generic="T extends MinimalSearchValue">
import { ref, onMounted, onBeforeUnmount } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
import {
  matchedResultsFilter,
  MinimalSearchValue,
  parseSearchValue
} from '@/components/common/search/helpers/match.js';

defineOptions({ name: 'SearchProperties' });

// Your existing props, emit, and other constants setup here ...

const opened = ref(false);
const searchDropdown = ref<HTMLElement | null>(null);

const focusDropdown = () => {
  opened.value = true;
};

const closeDropdown = () => {
  opened.value = false;
};

const checkClickOutside = (event) => {
  if (searchDropdown.value && !searchDropdown.value.contains(event.target as Node) && opened.value) {
    closeDropdown();
  }
};

onMounted(() => {
  document.addEventListener('click', checkClickOutside);
  window.addEventListener('keydown', (event) => {
    if (event.key === "Escape") {
      closeDropdown();
    }
  });
});

onBeforeUnmount(() => {
  document.removeEventListener('click', checkClickOutside);
  window.removeEventListener('keydown', closeDropdown);
});

// Remaining part of your script...
</script>
```

### Adjustments:

1. **Removed Unused Function**: The `focusEvent` function was removed.
2. **Click Handling**: The `checkClickOutside` function now checks if a click is outside the search dropdown and closes it if so.
3. **Keydown Event**: Listens for the Escape key to close the dropdown.

This setup keeps the dropdown open when interacting with items inside it and only closes it when clicking outside or pressing the Escape key, aligning with your initial requirements.

------
To keep the dropdown of your search bar open during interactions inside it and only close when the user clicks outside or presses ESC, you need to adjust the `@blur` event handling. Specifically, you should modify the `closeDropdown` method to better handle the focus state. Here's a revised version of your script with the necessary changes:

### Adjustments in `<template>`:

1. Remove `@blur="closeDropdown"` from the `<app-input>` component.
2. Adjust the `@blur` handler in the `div` containing the class "results" to check if the focus has moved outside the dropdown.

### Updated `<script setup>` section:

```javascript
<script setup lang="ts" generic="T extends MinimalSearchValue">
import { computed, ref, onMounted, onBeforeUnmount } from 'vue';
import AppInput from '@/components/common/formElements/input/Input.vue';
import AppIconButton from '@/components/common/icon/IconButton.vue';
import { useI18n } from 'vue-i18n';
import {
  matchedResultsFilter,
  MinimalSearchValue,
  parseSearchValue
} from '@/components/common/search/helpers/match.js';

defineOptions({ name: 'SearchProperties' });

// Your existing props, emit, and other constants setup here ...

const opened = ref(false);
const searchDropdown = ref<HTMLElement | null>(null);
const focusEvent = (e: FocusEvent) => {
  if (!searchDropdown.value || !(e.relatedTarget as Node).closest('.search-property-wrapper')) {
    closeDropdown();
  }
};

const focusDropdown = () => {
  opened.value = true;
};

const closeDropdown = () => {
  opened.value = false;
};

onMounted(() => {
  document.addEventListener('click', (event) => {
    if (!searchDropdown.value?.contains(event.target as Node) && opened.value) {
      closeDropdown();
    }
  });
  window.addEventListener('keydown', (event) => {
    if (event.key === "Escape") {
      closeDropdown();
    }
  });
});

onBeforeUnmount(() => {
  document.removeEventListener('click', closeDropdown);
  window.removeEventListener('keydown', closeDropdown);
});

// Remaining part of your script...
</script>
```

### Key Changes:

1. **Focus Handling**: Removed the `@blur` event from `app-input` and modified the blur handling in the dropdown `div` to check if the new focus is outside the search component. This prevents the dropdown from closing when interacting with items inside it.
2. **Click Outside Handling**: Added a global click listener to close the dropdown when clicking outside the component.
3. **Escape Key Handling**: Added a listener for the Escape key to close the dropdown.

This setup ensures the dropdown remains open when interacting with elements inside it but will close when the user clicks outside the dropdown area or presses the Escape key.



-------
The error message you're seeing in the backend suggests there's an issue with the handling of the product state, specifically related to determining if a product is frozen. The `NullPointerException` indicates that the method attempting to fetch the frozen state is receiving a `null` value, which it doesn't handle properly.

Here's a breakdown of the issue and steps to resolve it:

### Problem
- The method `ensureProductIsNotFrozen` in `ProductManagementService` is trying to invoke `booleanValue()` on a `Boolean` that is null. This happens because the method `isFrozen()` from `ExtendedProduct` returns null, which isn't expected by your code.

### Solutions
1. **Check for Null in Backend Service**: Modify the `ensureProductIsNotFrozen` method to handle possible null values. This will prevent your application from crashing when the product's frozen status isn't properly initialized or fetched. For example:
   ```java
   public void ensureProductIsNotFrozen(String productId) throws ProductFrozenException {
       ExtendedProduct product = productService.getProductById(productId);
       if (product != null && Boolean.TRUE.equals(product.isFrozen())) {
           throw new ProductFrozenException("Product is frozen and cannot be edited.");
       }
   }
   ```
   This code checks if `product` is not null and then explicitly checks if `isFrozen()` is true. This avoids the `NullPointerException` if `isFrozen()` is null.

2. **Review Product Fetching Logic**: Ensure that wherever you're fetching the product details, the product is fully initialized, including its frozen status. There might be a flaw in the logic where the product's attributes are set, or it might not be fetching the correct details from the database.

3. **Frontend Handling**: On the frontend, ensure you are handling these errors gracefully. When such an error occurs, provide feedback to the user that something went wrong, which you're already doing by logging "Failed to update track item." However, you might want to expand this to more user-friendly feedback using a UI element like a toast notification or error message display.

### Additional Note
Regarding the `submitEditRequests` method and the `props.productId` issue, it seems like `productId` isn't directly available in your component's props or context. Make sure you're either:

- Passing `productId` as a prop to each component that requires it.
- Fetching it from a global state or context that's accessible within the component, as seen in your other components using `productsStore.activeProductId`.

By fixing the null handling in the backend and ensuring the frontend properly uses `productId`, you should be able to resolve both the server-side error and the client-side integration issues.
