```typescript
// Add a new method to detach track item from its track
const detachTrackItemFromTrack = (xmiId: XmiId): void => {
  const trackItem = trackItems.value.find(item => item.xmiId === xmiId);
  if (trackItem) {
    // Detach the track item from its track
    trackItem.domain.track = null;
    eventService.emit(EventType.DomainObjectUpdated, xmiId);
  }
};

return {
  // existing state, getters, actions, mutations...
  detachTrackItemFromTrack,
};

```

```vue
<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from "vue";
import { useI18n } from "vue-i18n";

import { PropertyValue } from "@ebitoolmx/cbss-types";
import { XmiId } from "@ebitoolmx/eclipse-types";
import { extractXmiId } from "@ebitoolmx/ebitool-classic-types";
import { isNotEmpty } from "@ebitoolmx/predicates";

import { ToolType, ToolItemCategory } from "@/typings/tools.js";
import { SplitView } from "@/typings/splitView.js";
import { IndicatorType } from "@/typings/indicator.js";
import { PlainSelection } from "@/typings/selection/PlainSelection.js";
import { PlainHighlight } from "@/typings/highlight/PlainHighlight.js";

import AppDeletableIndicator from "@/components/common/indicator/DeletableIndicator.vue";
import AppIndicator from "@/components/common/indicator/Indicator.vue";
import AppPropertyContainer from "@/components/common/sidePanelElements/PropertyContainer.vue";
import AppSearchProperties from "@/components/common/search/SearchProperties.vue";
import AppIconButton from "@/components/common/icon/IconButton.vue";

import { eventService, EventType } from "@/services/event.js";

import { useInfrastructureStore } from "@/stores/infrastructure.js";
import { useDiagramStore } from "@/stores/diagram.js";
import { useEditorStore } from "@/stores/editor.js";
import { isMXObjectIdentifier } from "@/typings/selection";

defineOptions({ name: "TrackItemProperty" });

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
    category: "Unknown",
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

const groupedTrackItems = computed<Array<Array<[PropertyValue, number]>>>(
  () => {
    const groupedByMileage = props.value.reduce(
      (
        acc: Record<string, Array<[PropertyValue, number]>>,
        item: PropertyValue,
        i: number
      ) => {
        const domainObject = infrastructureStore.getTrackItem(
          item.objectIdentifier.id
        );
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
  }
);

const currentIds = computed<string[]>(() =>
  props.value.map((p) => extractXmiId(p.object.reference))
);

function getNodesToHighlight(propValues: PropertyValue[]): string[] {
  const nodes = propValues.map((value) => value.objectIdentifier.id);

  return nodes.filter(isNotEmpty);
}

function isAllValuesHighlighted(): boolean {
  if (editorStore.highlighted.isEmpty()) return false;

  const nodesToHighlight = getNodesToHighlight(props.value).filter(Boolean);

  // Returns true if all or more values are highlighted.
  // selecting an area group should indicate that all items are highlighted.
  // Even though the items belonging to the areas inside the group isn’t directly connected to the area group
  return nodesToHighlight.every((node) =>
    editorStore.highlighted.expandedIds.includes(node)
  );
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
    "change",
    currentIds.value.filter((id) => id !== xmiId)
  );
  editorStore.setHighlighted(
    new PlainHighlight(
      editorStore.highlighted.filter((id: string) => id !== xmiId)
    )
  );

  const newSelection = editorStore.selected.ids.filter((id) =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));
};

const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(
    new PlainHighlight([...editorStore.highlighted.expandedIds, xmiId])
  );
  emit("change", [...currentIds.value, xmiId]);
};

const addValue = (propValue: PropertyValue): void => {
  const includeXmiId = extractXmiId(propValue.object.reference);
  if (!currentIds.value.includes(includeXmiId)) addSelected(includeXmiId);
};

// Updated deleteValue to detach the item from the track
const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  // Call the infrastructureStore method to detach the track item from the track
  infrastructureStore.detachTrackItemFromTrack(excludeXmiId);
  deleteSelected(excludeXmiId);
};

const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: "MultiReferencePickerTool",
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

const isSplitViewSelected = (): boolean =>
  editorStore.isSplitViewShown(SplitView.Positions);

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
        editorStore.highlighted.filter(
          (id: string) => id !== propValue.objectIdentifier.id
        )
      )
    );
  }
};

onMounted(() =>
  eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle)
);

onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

```
