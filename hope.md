To resolve the issues and ensure proper handling of the new `RemoveTrackItem` event, we need to make updates in the following files: `event.ts`, `TrackItemProperty.vue`, and `Editor.vue`. Additionally, we need to ensure that we handle the event correctly within the editor store for selections and highlights.

Here's how you can achieve this:

### 1. Update `event.ts`
Add the new event type `RemoveTrackItem`.

```typescript
export enum EventType {
  // ... existing event types
  RemoveTrackItem = 'removeTrackItem',
}

type Events = {
  // ... existing events
  [EventType.RemoveTrackItem]: XmiId;
};

// ... existing createEventService function
```

### 2. Update `TrackItemProperty.vue`
Emit the `RemoveTrackItem` event when a track item is deleted.

```vue
<template>
  <!-- ... existing template code -->
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

  return nodesToHighlight.every(node => editorStore.highlighted.value.includes(node));
}

function isHighlighted(id: PropertyValue): boolean {
  const [nodeToHighlight] = getNodesToHighlight([id]);
  return editorStore.highlighted.value.includes(nodeToHighlight);
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
    new PlainHighlight(editorStore.highlighted.value.filter((id: string) => id !== xmiId))
  );

  const newSelection = editorStore.selected.value.ids.filter(id =>
    isMXObjectIdentifier(id) ? id.id : id !== xmiId
  );
  editorStore.setSelected(new PlainSelection(newSelection));

  // Emit the event to remove track item
  eventService.emit(EventType.RemoveTrackItem, xmiId);
};

const addSelected = (xmiId: XmiId): void => {
  editorStore.setHighlighted(new PlainHighlight([...editorStore.highlighted.value.expandedIds, xmiId]));
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
        editorStore.highlighted.value.filter((id: string) => id !== propValue.objectIdentifier.id)
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

  &::before,
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

### 3. Update `Editor.vue`
Subscribe to the `RemoveTrackItem` event and handle it accordingly.

```typescript
// ... existing imports

defineOptions({ name: 'InfrastructureEditor' });

useDiagramStore();

const { cbssPositionCorrectionSplittingEnabled } = useIsFeatureEnabled();

const gojsDiagram = useInjectDiagram(cbssDiagramKey);

useInfrastructure(cbssDiagramKey, false);
useInfrastructureTools(gojsDiagram);
useMultiUserCursors(gojsDiagram);

useInfrastructureReactivity(gojsDiagram);

const infrastructureStore = useInfrastructureStore();
const { selectToolItem } = useDiagramStore();
const editorStore = useEditorStore();
const preferencesStore = usePreferencesStore();
const productsStore = useProductsStore();

const onDiagramMounted = (target: string) => gojsDiagram.attachDiagram(target);

const canModifySelection = (): boolean => editorStore.isEditMode && !editorStore.selected.isEmpty();

const deleteSelected = async (): Promise<void> => {
  if (!preferencesStore.skipDeleteObjectDialog) {
    eventService.emit(EventType.OpenDialog, {
      dialogName: DialogNames.DeleteObjectConfirmation,
      options: {
        modal: true,
        props: {
          productId: productsStore.activeProductId,
          xmiIds: editorStore.selected.normalizedIds,
          deleteDomainObject: infrastructureStore.deleteDomainObject
        }
      }
    });
  } else {
    await infrastructureStore.deleteDomainObject({
      productId: productsStore.activeProductId,
      xmiIds: editorStore.selected.normalizedIds
    });
  }
};

const changeNotationDirection = (): void => {
  editorStore.selected.normalizedIds.forEach((xmiId: XmiId) => {
    const domainObject = infrastructureStore.getDomainObject(xmiId);
    const notationDirection = domainObject?.representation?.meta?.notationDirection;

    if (!notationDirection) return;

    const newDirection =
      notationDirection === BasicDirection.Nominal
        ? BasicDirection.Reverse
        : BasicDirection.Nominal;

    infrastructureStore.switchNotationDirection({
      productId: productsStore.activeProductId,
      xmiId,
      notationDirection: newDirection
    });
  });
};

const shortcutHandler = (shortcut: Shortcut) => {
  const isDelete = shortcut === Shortcut.Delete || shortcut === Shortcut.DeleteBackspace;
  const canChangeNotationDirection = shortcut === Shortcut.SwitchNotationDirection;

  if (
    isDelete &&
    canModifySelection() &&
    !isBaseSelection(editorStore.selected) &&
    !isModelMetaDataSelection(editorStore.selected)
  ) {
    deleteSelected();
  } else if (canChangeNotationDirection && canModifySelection()) {
    changeNotationDirection();
  } else if (shortcut === Shortcut.Escape) {
    selectToolItem(null);
  }
};

const onDiagramKeyDown = (): void => {
  const event = gojsDiagram.diagram.lastInput;
  if (event) {
    event.bubbles = true;
    event.handled = false;
  }
};

const clearTrackItemAdornments = (diagram: Diagram) => {
  const trackItems = diagram.findNodesByExample({ type: GoType.TrackItem });

  trackItems.each(trackItem => {
    trackItem.removeAdornment('trackItemOrder');
  });
};

const updateTrackItemAdornments = () => {
  clearTrackItemAdornments(gojsDiagram.diagram);

  const trackItems = gojsDiagram.diagram.findNodesByExample({ type: GoType.TrackItem });

  trackItems.each(trackItem => {
    const { trackId } = trackItem.data;
    const track = gojsDiagram.diagram.findLinkForKey(trackId);

    if (
      trackId &&
      track?.isSelected &&
      !isLineSelection(editorStore.selected) &&
      !isMainTrackSelection(editorStore.selected)
    ) {
      const adornment = trackItemsOrderTemplate(GraphObject.make);
      adornment.adornedObject = trackItem;
      trackItem?.addAdornment('trackItemOrder', adornment);
    }
  });
};

const onModelChanged = (event: ChangedEvent): void => {
  if (
    event.change === ChangedEvent.Property &&
    (event.propertyName === 'identity' || event.propertyName === 'label') &&
    event?.object?.group
  ) {
    const group = gojsDiagram.diagram.findPartForKey(event.object.group);
    if (group) {
      generateLabel(group as GroupWithData);
    }
  }

  if (
    event.change === ChangedEvent.Property &&
    (event.propertyName === 'trackItemIds' ||
      event.propertyName === 'angles' ||
      event.propertyName === 'location')
  ) {
    updateTrackItemAdornments();
  }
};

const onLinkedOrRelinked = (event: DiagramEvent): void => {
  (event.subject as Link).isSelected = false;
  const linkData = event.subject.data;

  const fromLeg = event.subject.fromNode.data.legs.find(
    (leg: DomainLeg) => leg.xmiId === linkData.fromPortId
  );
  const toLeg = event.subject.toNode.data.legs.find(
    (leg: DomainLeg) => leg.xmiId === linkData.toPortId
  );

  const track = infrastructureStore.getTrack(fromLeg.trackId ?? toLeg.trackId);

  if (fromLeg.hasTrack || toLeg.hasTrack) {
    infrastructureStore.relinkTrack({
      productId: productsStore.activeProductId,
      properties: {
        sourceLeg:
          fromLeg.xmiId === track?.domain.sourceLeg.reference
            ? linkData.fromPortId
            : linkData.toPortId,
        targetLeg:
          fromLeg.xmiId === track?.domain.sourceLeg.reference
            ? linkData.toPortId
            : linkData.fromPortId,
        xmiId: fromLeg.trackId ?? toLeg.trackId
      }
    });

    return;
  }

  infrastructureStore.addTrack({
    productId: productsStore.activeProductId,
    sourceLeg: linkData.fromPortId,
    targetLeg: linkData.toPortId
  });
};

const updateBendpoints = (link: Link): void => {
  const bendpoints: GoLocation[] = [];
  link.points.each(({ x, y }) => {
    bendpoints.push({ x, y });
  });

  infrastructureStore.reshapeTrack({
    productId: productsStore.activeProductId,
    xmiId: link.data.xmiId,
    bendpoints
  });
};

const onLinkReshaped = (event: DiagramEvent): void => {
  const link = event.subject as Link;
  updateBendpoints(link);
};

const updateAssociatedLinks = (part: Part): void => {
  if (part.data?.type === 'node') {
    part.data.legs.forEach((leg: { trackId: Key }) => {
      const link: Link | null = gojsDiagram.diagram.findLinkForKey(leg.trackId);
      if (link) updateBendpoints(link);
    });
  }
};

const onSelectionMoved = (event: DiagramEvent): void => {
  if (!editorStore.isEditMode) return;
  const updatedDomainBounds: UpdateBounds[] = [];

  const parts = event.subject as GoSet<Part>;
  parts.each(part => {
    const domainObject = infrastructureStore.getDomainObject(part?.data?.xmiId);
    const hasMoved =
      domainObject?.representation &&
      (part.location.x !== domainObject.representation?.location?.x ||
        part.location.y !== domainObject.representation?.location?.y);

    if (part && domainObject && hasMoved) {
      const movedObject = updateBounds(domainObject, { location: part.location });

      updatedDomainBounds.push({
        domainObject: movedObject,
        oldBounds: { location: domainObject?.representation?.location }
      });

      updateAssociatedLinks(part);
    }
  });

  if (!updatedDomainBounds.length) return;

  infrastructureStore.updateDomainObjectBounds({
    productId: productsStore.activeProductId,
    updatedDomainBounds
  });
};

const onPartResized = (event: DiagramEvent): void => {
  if (!editorStore.isEditMode) return;

  const part = event.subject as Part;
  const domainObject = infrastructureStore.getDomainObject(part?.data?.xmiId);

  if (part && domainObject) {
    const resizedObject = updateBounds(domainObject, {
      location: part.location,
      size: part.desiredSize
    });

    infrastructureStore.updateDomainObjectBounds({
      productId: productsStore.activeProductId,
      updatedDomainBounds: [
        {
          domainObject: resizedObject,
          oldBounds: {
            location: domainObject?.representation?.location,
            size: domainObject?.representation?.size
          }
        }
      ]
    });
  }
};

const drawTrackArrows = (
  tracks: Track[],
  links: Link[],
  calculateArrowDirections: (tracksMap: Map<Track, Link>) => Map<Link, BasicDirection>
): void => {
  const tracksMap = new Map(zip(tracks, links));
  const directions = calculateArrowDirections(tracksMap);

  drawArrows(directions);
};

const addTrackArrows = (diagram: Diagram, domainObject: MainTrack |

 Line) => {
  const tracks = domainObject.domain.tracks
    ?.map(trackRef => infrastructureStore.getDomainObject(trackRef.reference))
    .filter(isNotEmpty) as Track[];

  const links = tracks.map(o => diagram.findLinkForKey(o.xmiId)).filter(isNotEmpty);

  const calculateArrowDirections = getTrackDirectionCalculator(
    infrastructureStore.getNodeFromLeg,
    isMainTrack(domainObject),
    isMainTrack(domainObject) ? domainObject.domain.travellingDirection : undefined
  );

  drawTrackArrows(tracks, links, calculateArrowDirections);
};

const clearTrackArrows = (diagram: Diagram) => {
  diagram.links.each(link => {
    link.elements.filter(go => go.name.startsWith(ArrowNameBase)).each(go => link.remove(go));
  });
};

const updateTrackArrows = () => {
  clearTrackArrows(gojsDiagram.diagram);
  if (isLineSelection(editorStore.selected) || isMainTrackSelection(editorStore.selected)) {
    addTrackArrows(gojsDiagram.diagram, editorStore.selected.domainObject);
  } else if (
    isLineHighlight(editorStore.highlighted) ||
    isMainTrackHighlight(editorStore.highlighted)
  ) {
    addTrackArrows(gojsDiagram.diagram, editorStore.highlighted.domainObject);
  }
};

const onSelectionChanged = () => {
  updateTrackArrows();
  updateTrackItemAdornments();
};

const onHighlightedChanged = () => {
  updateTrackArrows();
};

const removeTrackItem = (excludeXmiId: XmiId) => {
  const trackItems = gojsDiagram.diagram.findNodesByExample({ type: GoType.TrackItem });

  trackItems.each(trackItem => {
    if (trackItem.data.xmiId === excludeXmiId) {
      gojsDiagram.diagram.remove(trackItem);
    }
  });
};

onMounted(() => {
  eventService.on(EventType.Shortcut, shortcutHandler);
  eventService.on(EventType.RemoveTrackItem, removeTrackItem);

  gojsDiagram.diagram.commandHandler.doKeyDown = onDiagramKeyDown;
  gojsDiagram.diagram.addModelChangedListener(onModelChanged);
  gojsDiagram.diagram.addDiagramListener('LinkDrawn', onLinkedOrRelinked);
  gojsDiagram.diagram.addDiagramListener('LinkReshaped', onLinkReshaped);
  gojsDiagram.diagram.addDiagramListener('SelectionMoved', onSelectionMoved);
  gojsDiagram.diagram.addDiagramListener('PartResized', onPartResized);
});

onBeforeUnmount(() => {
  eventService.off(EventType.Shortcut, shortcutHandler);
  eventService.off(EventType.RemoveTrackItem, removeTrackItem);

  gojsDiagram.diagram.removeModelChangedListener(onModelChanged);
  gojsDiagram.diagram.removeDiagramListener('LinkDrawn', onLinkedOrRelinked);
  gojsDiagram.diagram.removeDiagramListener('LinkReshaped', onLinkReshaped);
  gojsDiagram.diagram.removeDiagramListener('SelectionMoved', onSelectionMoved);
  gojsDiagram.diagram.removeDiagramListener('PartResized', onPartResized);
});

watch(() => editorStore.selected, onSelectionChanged);
watch(() => editorStore.highlighted, onHighlightedChanged);
watch(
  () => editorStore.isEditMode,
  isEdit => {
    if (isEdit) {
      clearCurrentLineAndItemGroups(
        productsStore.activeProductId,
        infrastructureStore.site ?? null
      );

      if (cbssPositionCorrectionSplittingEnabled.value) {
        splitPositionCorrectionsWithMultipleTracks(
          infrastructureService,
          productsStore.activeProductId,
          infrastructureStore.positionCorrections,
          infrastructureStore.getTrackPoints,
          infrastructureStore.siteIdentifier?.id ?? ''
        );
      }
    }
  }
);
```

### 4. Update the Editor Store (`editor.ts`)

Ensure that `highlighted` and `selected` have the correct structure and initialize them properly.

```typescript
import { acceptHMRUpdate, defineStore } from 'pinia';
import { ref, computed, toRaw } from 'vue';

import { XmiId } from '@ebitoolmx/eclipse-types';

import { EditorType } from '@/typings/store.js';
import { Panel } from '@/typings/sidePanel.js';
import { SplitView, SplitViewOptions } from '@/typings/splitView.js';
import { EditorMode } from '@/typings/editor.js';
import { Theme } from '@/typings/theme.js';
import { PlainSelection } from '@/typings/selection/PlainSelection.js';
import { PlainHighlight } from '@/typings/highlight/PlainHighlight.js';

import { metricsService } from '@/services/metrics.js';
import { loggingService } from '@/services/logging.js';
import { isHighlight, Highlight } from '@/typings/highlight/index.js';
import { isSelection, Selection } from '@/typings/selection/index.js';

export const useEditorStore = defineStore('editor', () => {
  /* State */
  const forcedTheme = ref<Theme | null>(null);
  const requiresCloud = ref(true);
  const editorType = ref<EditorType | null>(null);
  const isSearchOpen = ref(false);
  const sidePanel = ref<Panel>(Panel.Blank);
  const lastSidePanelState = ref<Panel>(Panel.Properties);
  const secondarySidePanel = ref<Panel>(Panel.Blank);
  const splitViewOptions = ref<SplitViewOptions[]>([]);
  const selectedSplitView = ref<SplitView | null>(null);
  const editorMode = ref<EditorMode>(EditorMode.NoProduct);
  const _selected = ref<Selection>(new PlainSelection());
  const _highlighted = ref<Highlight>(new PlainHighlight());
  const _focusable = ref<boolean>(false);

  /* Getters */
  const isEditMode = computed<boolean>(() => editorMode.value === EditorMode.Edit);
  const isViewMode = computed<boolean>(() => editorMode.value === EditorMode.View);
  const isReadMode = computed<boolean>(() => editorMode.value === EditorMode.Read);
  const selected = computed<Selection>(() => _selected.value);
  const highlighted = computed<Highlight>(() => _highlighted.value);
  const isFocusable = computed<boolean>(() => _focusable.value);
  const splitViews = computed<SplitView[]>(() => splitViewOptions.value.map(sv => sv.type));

  const isSplitViewShown = (v: SplitView): boolean => selectedSplitView.value === v;

  /* Actions */
  const setFocusable = (value: boolean) => {
    _focusable.value = value;
  };

  const setRequiresCloud = (value: boolean) => {
    requiresCloud.value = value;
  };

  const setEditorType = (value: EditorType) => {
    editorType.value = value;
  };

  const setEditorMode = (mode: EditorMode) => {
    metricsService.trackEvent('editorModeChanged', { newMode: mode, oldMode: editorMode.value });
    editorMode.value = mode;
  };

  const showSearch = () => {
    isSearchOpen.value = true;
  };

  const hideSearch = () => {
    isSearchOpen.value = false;
  };

  const setSidePanel = (panel: Panel) => {
    sidePanel.value = panel;
    if (panel !== Panel.Blank) lastSidePanelState.value = panel;
  };

  const setSecondarySidePanel = (panel: Panel) => {
    secondarySidePanel.value = panel;
  };

  const setSplitView = (view: SplitView) => {
    splitViewOptions.value = [{ type: view }];
    selectedSplitView.value = view;
    metricsService.trackEvent('splitViewChanged', { view });
  };

  const closeSplitView = () => {
    splitViewOptions.value = [];
    selectedSplitView.value = null;
    metricsService.trackEvent('splitViewChanged', { view: 'hidden' });
  };

  const addSplitView = (view: SplitView, title?: string) => {
    if (!splitViews.value.includes(view)) {
      splitViewOptions.value = [
        ...splitViewOptions.value,
        title ? { type: view, title } : { type: view }
      ];
    } else if (title) {
      splitViewOptions.value = splitViewOptions.value.map(svo => {
        return svo.type === view ? { ...svo, title } : svo;
      });
    }
    selectedSplitView.value = view;
  };

  const removeSplitView = (viewToRemove: SplitView) => {
    splitViewOptions.value = splitViewOptions.value.filter(view => view.type !== viewToRemove);
    if (selectedSplitView.value === viewToRemove) {
      selectedSplitView.value = splitViews.value?.[0] ?? null;
    }
  };

  const selectSplitView = (viewToSelect: SplitView) => {
    selectedSplitView.value = viewToSelect;
  };

  const toggleSplitView = (viewToToggle: SplitView) => {
    if (splitViews.value.includes(viewToToggle)) {
      removeSplitView(viewToToggle);
    } else {
      setSplitView(viewToToggle);
    }
  };

  const setSelected = (newSelected: Selection) => {
    if (isSelection(newSelected)) {
      _selected.value = newSelected;
    } else {
      loggingService.warn(
        'setSelected requires a Selection class to be passed to it. Selection has not been changed.'
      );
    }
  };

  const setHighlighted = (newHighlighted: Highlight) => {
    if (isHighlight(newHighlighted)) {
      _highlighted.value = newHighlighted;
    } else {
      loggingService.warn(
        'setHighlighted requires a Highlight class to be passed to it. Highlight has not been changed.'
      );
    }
  };

  const toggleHighlighted = (newHighlighted:

 Highlight) => {
    if (newHighlighted.isEmpty() || _highlighted.value.isEmpty()) {
      _highlighted.value = newHighlighted;
    } else if (newHighlighted.isSameHighlightAs(_highlighted.value)) {
      _highlighted.value = new PlainHighlight();
    } else {
      _highlighted.value = newHighlighted;
    }
  };

  const clearSelection = () => {
    setSelected(new PlainSelection());
    setHighlighted(new PlainHighlight());
  };

  const domainObjectsDeleted = (xmiIds: XmiId[]) => {
    xmiIds.forEach(xmiId => {
      if (_selected.value.includes(xmiId)) clearSelection();
    });
  };

  const $reset = () => {
    forcedTheme.value = null;
    requiresCloud.value = true;
    editorType.value = null;
    editorMode.value = EditorMode.NoProduct;
    isSearchOpen.value = false;
    splitViewOptions.value = [];
    selectedSplitView.value = null;
    _selected.value = new PlainSelection();
    _highlighted.value = new PlainHighlight();
  };

  return {
    requiresCloud,
    forcedTheme,
    editorType,
    editorMode,
    isEditMode,
    isViewMode,
    isReadMode,
    isSearchOpen,
    sidePanel,
    lastSidePanelState,
    secondarySidePanel,
    splitViewOptions,
    splitViews,
    selectedSplitView,
    selected,
    highlighted,
    isSplitViewShown,
    toggleHighlighted,
    clearSelection,
    domainObjectsDeleted,
    setRequiresCloud,
    setEditorType,
    setEditorMode,
    showSearch,
    hideSearch,
    setSidePanel,
    setSecondarySidePanel,
    setSplitView,
    closeSplitView,
    addSplitView,
    removeSplitView,
    selectSplitView,
    toggleSplitView,
    setSelected,
    setHighlighted,
    setFocusable,
    isFocusable,
    $reset
  };
});

if (import.meta.hot) {
  import.meta.hot.accept(acceptHMRUpdate(useEditorStore, import.meta.hot));
}
```

These updates should ensure that the `RemoveTrackItem` event is properly handled and that track items are removed correctly.

-----
### Analysis and Interaction Summary

#### **Editor.vue (InfrastructureEditor.vue)**

1. **GoJS Diagram Setup**:
    - The GoJS diagram is set up with `useInjectDiagram(cbssDiagramKey)`.
    - It initializes and updates the diagram based on interactions (`onDiagramMounted`, `onModelChanged`, etc.).

2. **Highlight and Selection**:
    - Functions like `clearTrackItemAdornments`, `updateTrackItemAdornments`, `onSelectionChanged`, and `onHighlightedChanged` handle the highlighting and selection changes in the diagram.

3. **Event Handling**:
    - It listens to various events (`LinkDrawn`, `LinkReshaped`, `SelectionMoved`, `PartResized`) to update the state.

4. **Store Interaction**:
    - Utilizes `infrastructureStore`, `editorStore`, `productsStore`, etc., for managing state and performing actions such as deleting, adding, or updating domain objects.

#### **TrackItemProperty.vue**

1. **Track Item Management**:
    - Handles adding, deleting, and highlighting track items through methods like `addValue`, `deleteValue`, and `selectItemGroup`.

2. **Highlighting and Selection**:
    - Uses `eventService` to emit events for highlighting (`EventType.CenterHighlighted`).

3. **Computed Properties**:
    - `currentIds`, `groupedTrackItems`, `isEditing` manage and track the state of track items.

#### **infrastructure.ts**

1. **State Management**:
    - Manages state for tracks, track items, nodes, detectors, signal equipment rooms, position corrections, notes, lines, item groups, area groups, main tracks, line profiles, site, and model metadata.

2. **Getters and Actions**:
    - Provides getters for retrieving domain objects, track items, nodes, lines, etc.
    - Includes actions for adding, deleting, and updating domain objects, subscribing to updates, and handling WebSocket messages.

3. **Events**:
    - Utilizes `eventService` to emit and listen for events such as `DomainObjectUpdated` and `DomainObjectDeleted`.

#### **editor.ts**

1. **State Management**:
    - Manages editor state including selected and highlighted items, editor mode, side panels, split views, etc.

2. **Actions**:
    - Provides actions for setting selected and highlighted items, toggling split views, and handling domain object deletions.

3. **Events**:
    - Tracks events such as editor mode changes and split view changes using `metricsService`.

#### **event.js**

1. **Event Service**:
    - Implements a global event bus (`mitt`) for emitting and listening to events anywhere in the app.

2. **Event Types**:
    - Defines various event types (`ScaleToFit`, `CenterDiagram`, `Shortcut`, `OpenDialog`, etc.).

### Connection and Interaction

- **Highlighting and Selection**:
    - When a track item is highlighted or selected in `TrackItemProperty.vue`, it emits events (`EventType.CenterHighlighted`) using `eventService`.
    - `Editor.vue` listens to these events and updates the GoJS diagram accordingly.

- **State Management**:
    - Both `Editor.vue` and `TrackItemProperty.vue` interact with the Vuex store (`infrastructureStore`, `editorStore`) to manage the state of track items and their highlighting/selection.

- **Event Handling**:
    - `eventService` is used for global communication between components, ensuring that actions in one component (e.g., deleting a track item) are reflected in others.

### Next Steps

To implement the desired feature of removing a track item from `TrackItemProperty.vue` without deleting it from the GoJS diagram, follow these steps:

1. **Update `TrackItemProperty.vue`**:
    - Modify the `deleteValue` method to emit an event or update the store to remove the track item from the `TrackItemProperty` list but not from the diagram.

2. **Handle Removal in `Editor.vue`**:
    - Listen for the event emitted by `TrackItemProperty.vue` in `Editor.vue` and update the GoJS diagram to reflect the removal of the track item from the selection/highlight list.

### Implementation

#### **TrackItemProperty.vue**

Update the `deleteValue` method to emit an event for removal:

```javascript
const deleteValue = (propValue: PropertyValue): void => {
  const excludeXmiId = extractXmiId(propValue.object.reference);
  deleteSelected(excludeXmiId);
  
  // Emit an event to remove the track item from the property list
  eventService.emit(EventType.RemoveTrackItem, excludeXmiId);
};
```

#### **Editor.vue**

Listen for the `RemoveTrackItem` event and update the diagram:

```javascript
onMounted(() => {
  eventService.on(EventType.RemoveTrackItem, (xmiId: string) => {
    const trackItem = infrastructureStore.getTrackItem(xmiId);
    if (trackItem) {
      // Remove the track item from the selection/highlight list
      editorStore.setHighlighted(
        new PlainHighlight(editorStore.highlighted.filter((id: string) => id !== xmiId))
      );
      
      const newSelection = editorStore.selected.ids.filter(id => id !== xmiId);
      editorStore.setSelected(new PlainSelection(newSelection));
      
      // Update the track item adornments
      updateTrackItemAdornments();
    }
  });
  
  // Existing event listeners...
  eventService.on(EventType.Shortcut, shortcutHandler);
  gojsDiagram.diagram.commandHandler.doKeyDown = onDiagramKeyDown;
  gojsDiagram.diagram.addModelChangedListener(onModelChanged);
  gojsDiagram.diagram.addDiagramListener('LinkDrawn', onLinkedOrRelinked);
  gojsDiagram.diagram.addDiagramListener('LinkReshaped', onLinkReshaped);
  gojsDiagram.diagram.addDiagramListener('SelectionMoved', onSelectionMoved);
  gojsDiagram.diagram.addDiagramListener('PartResized', onPartResized);
});
```

### Next Steps for Further Investigation

1. **Test the Changes**:
    - Test the updated functionality to ensure that track items are correctly removed from the `TrackItemProperty` list without being deleted from the GoJS diagram.

2. **Additional Features**:
    - Consider adding features like undo/redo for track item removal or providing a confirmation dialog before removal.

**Suggestions for the next user turn**:

**a.** Run the updated Vue components in the development environment to test the new feature.

**b.** Add unit tests for the `deleteValue` method in `TrackItemProperty.vue` and the event handling in `Editor.vue`.

-------------
Yes, you can pass the `allModels` and `syncedModels` data to the `Spinner.vue` component from `useSync.ts`. Since `useSync.ts` already handles the WebSocket subscription and synchronization status, you can modify the `toastService.display` calls to include this data as additional properties.

### Steps to Pass Data from `useSync.ts` to `Spinner.vue`

1. **Update `toastService.display` in `useSync.ts`:**
   - Add `allModels` and `syncedModels` to the `toastService.display` call when creating the spinner toast.

2. **Modify `Spinner.vue` to Accept New Props:**
   - Update `Spinner.vue` to accept `allModels` and `syncedModels` as props.

3. **Update Toast Service Handling:**
   - Ensure the toast service can handle and pass these new props to the `Spinner.vue` component.

### High-Level Steps:

1. **Update `useSync.ts` for Spinner Toast:**
   - When creating the spinner toast, include `allModels` and `syncedModels` in the properties passed to `toastService.display`.

   ```javascript
   if ((syncProcessing || syncInitialising) && !syncProcessingId.value) {
     syncProcessingId.value = toastService.display(ToastType.Spinner, {
       type: 'default',
       title: 'sync.toaster.requested',
       titleVars: {
         syncedBy: newStatus.synchronizationProgress?.syncedBy
       },
       allModels: newStatus.synchronizationProgress?.allModels,
       syncedModels: newStatus.synchronizationProgress?.syncedModels,
       dismissable: true
     });
   }
   ```

2. **Modify `Spinner.vue` to Accept Props:**
   - Update `Spinner.vue` to accept `allModels` and `syncedModels` as props.

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
       <div v-if="allModels && syncedModels">
         <p>All Models: {{ allModels.join(', ') }}</p>
         <p>Synced Models: {{ syncedModels.join(', ') }}</p>
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
       allModels?: string[];
       syncedModels?: string[];
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

   onMounted(() => {
     toastService.on(ToastEvent.Dismiss, onDismiss);
   });

   onBeforeUnmount(() => {
     toastService.off(ToastEvent.Dismiss, onDismiss);
   });
   </script>

   <style scoped>
   /* styles */
   </style>
   ```

3. **Ensure Toast Service Can Handle New Props:**
   - Verify that the toast service can pass the `allModels` and `syncedModels` props to `Spinner.vue`.

By following these steps, you can ensure that `Spinner.vue` receives the necessary synchronization data (`allModels` and `syncedModels`) directly from `useSync.ts` via the toast service. This approach ensures real-time synchronization data is displayed in the spinner toast for all users, even if they are on different networks and devices.


-----------

To ensure that the Toast Service can handle the new props (`allModels` and `syncedModels`) and pass them to the `Spinner.vue` component, you'll need to update the `ToastService` to pass these properties correctly and modify the `Toaster.vue` component to accept and pass these properties to the spinner component.

### Step-by-Step Solution:

1. **Update `ToastService` to Include New Props:**
   - Modify the `displayWithId` method to handle additional props for `allModels` and `syncedModels`.

   ```typescript
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

   public display(
     type: ToastType,
     props: Record<string, unknown> = {},
     timeout: number | undefined = undefined
   ): symbol {
     return this.displayWithId(Symbol('toast'), type, props, timeout);
   }
   ```

2. **Update `Spinner.vue` to Accept New Props:**
   - Ensure that `Spinner.vue` can receive and display the new properties.

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
       <div v-if="allModels && syncedModels">
         <p>All Models: {{ allModels.join(', ') }}</p>
         <p>Synced Models: {{ syncedModels.join(', ') }}</p>
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
       allModels?: string[];
       syncedModels?: string[];
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

   onMounted(() => {
     toastService.on(ToastEvent.Dismiss, onDismiss);
   });

   onBeforeUnmount(() => {
     toastService.off(ToastEvent.Dismiss, onDismiss);
   });
   </script>

   <style scoped>
   /* styles */
   </style>
   ```

3. **Update `Toaster.vue` to Handle New Props:**
   - Ensure that `Toaster.vue` can handle the new properties and pass them to the components.

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
   /* styles */
   </style>
   ```

This ensures that the `ToastService` can handle and pass the `allModels` and `syncedModels` props to the `Spinner.vue` component through `Toaster.vue`.

Do you have any specific test cases or scenarios in mind to ensure that the spinner toast displays the correct synchronization data?
---------------------


Based on the logs, it appears that `allModels` and `syncedModels` data are being passed inconsistently. Since `sync.vue` achieves live updates, let's ensure that the spinner toast can also update reactively.

Here are the steps to ensure reactive updates for `Spinner.vue`:

### 1. Ensure Reactive Data Handling in `useSync.ts`

Make sure `syncStatusSubscription` is correctly set up as a reactive reference and updates its state:

```typescript
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
        allModels: newStatus.synchronizationProgress?.allModels || [],
        syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
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
```

### 2. Ensure `Toaster.vue` Can Handle Dynamic Updates

Make sure `Toaster.vue` is correctly set up to pass dynamic props to the toast components. It looks fine, but ensure no issues in prop passing.

### 3. Make `Spinner.vue` Reactive to Props

Ensure `Spinner.vue` is reactive and updates whenever `allModels` or `syncedModels` props change.

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
    <div v-if="allModels && allModels.length && syncedModels && syncedModels.length">
      <p>All Models: {{ allModels.join(', ') }}</p>
      <p>Synced Models: {{ syncedModels.join(', ') }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, watch } from 'vue';
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
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    allModels: [],
    syncedModels: []
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

watch(
  () => props.allModels,
  (newModels) => {
    console.log('Updated allModels:', newModels);
  },
  { immediate: true }
);

watch(
  () => props.syncedModels,
  (newModels) => {
    console.log('Updated syncedModels:', newModels);
  },
  { immediate: true }
);
</script>

<style scoped>
/* styles */
</style>
```

### 4. Synchronize State Across Components

Ensure the state is synchronized and consistent across all components:

1. **Ensure `useSync.ts` Always Sends Data:**
   - Make sure the data is always being sent and logged correctly.

2. **React to Prop Changes in `Spinner.vue`:**
   - Use watchers in `Spinner.vue` to reactively update and log changes to props.

### Summary

- **Reactive Data Handling:**
  - Ensure `useSync.ts` properly watches and passes reactive data.
- **Prop Handling:**
  - Ensure `Spinner.vue` is set up to reactively update based on prop changes.
- **Consistency:**
  - Use logging to ensure consistency and trace data flow.

By following these steps, you can ensure that `Spinner.vue` correctly receives and updates with the `allModels` and `syncedModels` data consistently. If you continue to see issues, check for any potential race conditions or timing problems that might affect the data flow.

-----


Based on the logs, it appears that `allModels` and `syncedModels` data are being passed inconsistently. Since `sync.vue` achieves live updates, let's ensure that the spinner toast can also update reactively.

Here are the steps to ensure reactive updates for `Spinner.vue`:

### 1. Ensure Reactive Data Handling in `useSync.ts`

Make sure `syncStatusSubscription` is correctly set up as a reactive reference and updates its state:

```typescript
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
        allModels: newStatus.synchronizationProgress?.allModels || [],
        syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
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
```

### 2. Ensure `Toaster.vue` Can Handle Dynamic Updates

Make sure `Toaster.vue` is correctly set up to pass dynamic props to the toast components. It looks fine, but ensure no issues in prop passing.

### 3. Make `Spinner.vue` Reactive to Props

Ensure `Spinner.vue` is reactive and updates whenever `allModels` or `syncedModels` props change.

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
    <div v-if="allModels && allModels.length && syncedModels && syncedModels.length">
      <p>All Models: {{ allModels.join(', ') }}</p>
      <p>Synced Models: {{ syncedModels.join(', ') }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, watch } from 'vue';
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
  }>(),
  {
    titleVars: () => ({}),
    dismissable: false,
    allModels: [],
    syncedModels: []
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

watch(
  () => props.allModels,
  (newModels) => {
    console.log('Updated allModels:', newModels);
  },
  { immediate: true }
);

watch(
  () => props.syncedModels,
  (newModels) => {
    console.log('Updated syncedModels:', newModels);
  },
  { immediate: true }
);
</script>

<style scoped>
/* styles */
</style>
```

### 4. Synchronize State Across Components

Ensure the state is synchronized and consistent across all components:

1. **Ensure `useSync.ts` Always Sends Data:**
   - Make sure the data is always being sent and logged correctly.

2. **React to Prop Changes in `Spinner.vue`:**
   - Use watchers in `Spinner.vue` to reactively update and log changes to props.

### Summary

- **Reactive Data Handling:**
  - Ensure `useSync.ts` properly watches and passes reactive data.
- **Prop Handling:**
  - Ensure `Spinner.vue` is set up to reactively update based on prop changes.
- **Consistency:**
  - Use logging to ensure consistency and trace data flow.

By following these steps, you can ensure that `Spinner.vue` correctly receives and updates with the `allModels` and `syncedModels` data consistently. If you continue to see issues, check for any potential race conditions or timing problems that might affect the data flow.

------

To ensure that the spinner toast actively reflects changes in synchronization status, similar to `Sync.vue`, we need to ensure that the toast updates its state dynamically in response to changes in the synchronization progress. This means the toast should be able to react to multiple state updates and not just a one-time initialization.

### Steps to Ensure Active Reflection of Changes

1. **Emit Updates Continuously:**
   Ensure that updates to the synchronization status are continuously emitted and handled by the `ToastService`.

2. **Ensure ToastService Can Handle Updates:**
   Make sure `ToastService` can handle multiple updates for the same toast instance.

3. **Ensure Toaster.vue and Spinner.vue React to Updates:**
   Ensure `Toaster.vue` and `Spinner.vue` are set up to reactively update when they receive new props.

### Detailed Steps

1. **Emit Updates Continuously in `useSync.ts`:**
   
   Make sure you emit toast updates whenever there is a change in synchronization status, not just once.

   ```typescript
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

       console.log('Sync Status Updated:', newStatus.synchronizationProgress);

       if (syncProcessing || syncInitialising) {
         toastService.update(syncProcessingId.value, {
           type: 'default',
           title: 'sync.toaster.requested',
           titleVars: {
             syncedBy: newStatus.synchronizationProgress?.syncedBy
           },
           allModels: newStatus.synchronizationProgress?.allModels || [],
           syncedModels: newStatus.synchronizationProgress?.syncedModels || [],
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
   ```

2. **Extend `ToastService` to Handle Updates:**
   
   Modify the `ToastService` to support updates to existing toasts.

   ```typescript
   export class ToastService {
     protected toastComponents = new Map<ToastType, DefineComponent>();

     protected events: Emitter<Events> = mitt<Events>();

     public constructor(protected loggingService: LoggingService) {}

     public registerToast(type: ToastType, toast: any) {
       this.toastComponents.set(type, toast);
     }

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
       this.loggingService.debug(`Displaying toast of type '${type}'`, properties);

       return id;
     }

     public update(id: symbol, props: Record<string, unknown> = {}) {
       const toast = this.toastComponents.get(ToastType.Spinner);
       if (!toast) return;

       this.events.emit(ToastEvent.Display, { id, component: toast, properties: props });
       this.loggingService.debug(`Updating toast of type '${ToastType.Spinner}'`, props);
     }

     public display(
       type: ToastType,
       props: Record<string, unknown> = {},
       timeout: number | undefined = undefined
     ): symbol {
       return this.displayWithId(Symbol('toast'), type, props, timeout);
     }

     public dismiss(id: symbol): void {
       this.events.emit(ToastEvent.Dismiss, id);
     }

     public closeAll(): void {
       this.events.emit(ToastEvent.CloseAll);
     }

     public on<Type extends ToastEvent.CloseAll>(type: Type, callback: Function): void;
     public on<Type extends ToastEvent.Dismiss>(type: Type, callback: Function): void;
     public on<Type extends ToastEvent.Display>(type: Type, callback: (toast: Toast) => void): void;
     public on<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
       this.events.on(type, callback);
     }

     public off<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
       this.events.off(type, callback);
     }
   }

   export const toastService = new ToastService(commonLoggingService);
   ```

3. **Ensure `Toaster.vue` Handles Updates Correctly:**

   Ensure `Toaster.vue` listens for updates and passes them to `Spinner.vue`.

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
     const existingToastIndex = displayedToasts.value.findIndex(t => t.id === toast.id);
     if (existingToastIndex >= 0) {
       displayedToasts.value[existingToastIndex] = { ...displayedToasts.value[existingToastIndex], ...toast };
       displayedToasts.value = [...displayedToasts.value]; // force reactivity
     } else {
       displayedToasts.value = [...displayedToasts.value, toast];
     }

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
   /* styles */
   </style>
   ```

4. **Ensure `Spinner.vue` Reactively Updates:**

   Make sure `Spinner.vue` is set up to reactively update when props change.

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
       <div v-if="allModels.length || syncedModels.length">
         <p>All Models: {{ allModels.join(', ') }}</p>
         <p>Synced Models: {{ syncedModels.join(', ') }}</p>
       </div>
     </div>
   </template>

   <script setup lang="ts">
   import { onBeforeUnmount, onMounted, watch } from 'vue';
   import AppIconButton from '@/components/common/icon/IconButton.vue';
   import AppHorizontalSpinner from '@/components/common/spinner/HorizontalSpinner.vue';
   import

 { ToastEvent, toastService } from '@/services/toast.js';
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
     }>(),
     {
       titleVars: () => ({}),
       dismissable: false,
       allModels: [],
       syncedModels: []
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

   watch(
     () => props.allModels,
     (newModels) => {
       console.log('Updated allModels:', newModels);
     },
     { immediate: true }
   );

   watch(
     () => props.syncedModels,
     (newModels) => {
       console.log('Updated syncedModels:', newModels);
     },
     { immediate: true }
   );
   </script>

   <style scoped>
   /* styles */
   </style>
   ```

By ensuring the `ToastService` can handle updates and `Toaster.vue` and `Spinner.vue` reactively update with the new data, you should achieve the desired active reflection of synchronization status in the spinner toast.
