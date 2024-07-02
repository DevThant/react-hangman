Thanks for the context and screenshots. The issue seems to be related to the fact that the picker tool or any cursor-interactive tool is interfering with the scroll and input functionalities within the side panel. 

Here's a systematic approach to diagnose and fix this issue:

### Diagnosis

1. **Tool Activation and Deactivation:**
   - Ensure that the picker tool is properly deactivated when it's not in use.
   - Check if the tool is capturing mouse events even when the cursor is over the side panel.

2. **Event Propagation:**
   - Verify if the events are properly propagated to the side panel when the picker tool is active.

### Potential Solutions

1. **Event Propagation Control:**
   - Modify the event handling in the picker tool to ensure that events are not captured when the cursor is over the side panel.

2. **Tool Deactivation:**
   - Ensure the picker tool is deactivated when the cursor is over the side panel, and reactivated when it moves back to the diagram.

### Implementation Steps

1. **Update the `MultiReferenceClickPickingTool` to Ignore Events Over the Side Panel:**

```typescript
// MultiReferenceClickPickingTool.ts
import { Panel, Point } from 'gojs';

export class MultiReferenceClickPickingTool extends MxClickPickingTool {
  // ... existing code

  private isCursorOverSidePanel(): boolean {
    const cursorPoint = this.diagram.lastInput.viewPoint;
    const sidePanelElement = document.querySelector('.side-panel');
    if (sidePanelElement) {
      const rect = sidePanelElement.getBoundingClientRect();
      return (
        cursorPoint.x >= rect.left &&
        cursorPoint.x <= rect.right &&
        cursorPoint.y >= rect.top &&
        cursorPoint.y <= rect.bottom
      );
    }
    return false;
  }

  public doMouseDown(): void {
    if (this.isCursorOverSidePanel()) {
      this.stopTool();
      return;
    }
    super.doMouseDown();
  }

  public doMouseMove(): void {
    if (this.isCursorOverSidePanel()) {
      this.stopTool();
      return;
    }
    super.doMouseMove();
  }

  public doMouseUp(): void {
    if (this.isCursorOverSidePanel()) {
      this.stopTool();
      return;
    }
    super.doMouseUp();
  }
}
```

2. **Ensure Proper Tool Deactivation and Reactivation:**

Update the `TrackItemProperty.vue` to ensure proper deactivation and reactivation of the tool when the cursor is over the side panel.

```vue
<template>
  <!-- existing template code -->
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount } from 'vue';
import { useDiagramStore } from '@/stores/diagram';
import { eventService, EventType } from '@/services/event';

const diagramStore = useDiagramStore();
const isEditing = computed(() => {
  return (
    diagramStore.selectedToolItem?.category === props.category &&
    diagramStore.selectedToolItem.toolType === ToolType.PickerTool
  );
});

const toggleEdit = () => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool',
      id: props.id
    });
    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);
    }
    return;
  }
  diagramStore.selectToolItem(null);
};

const handleSidePanelToggle = () => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
};

onMounted(() => {
  eventService.on(EventType.ToggleSidePanel, handleSidePanelToggle);
});

onBeforeUnmount(() => {
  if (diagramStore.selectedToolItem?.toolType === ToolType.PickerTool) {
    diagramStore.selectToolItem(null);
  }
});
</script>

<style scoped>
/* existing styles */
</style>
```

This code ensures that when the picker tool is active and the cursor moves over the side panel, the tool is deactivated. When the cursor moves back to the diagram, the tool is reactivated.

### Testing

1. Activate the picker tool and try scrolling within the side panel.
2. Ensure that the side panel scroll and input functionalities work as expected.
3. Verify that the picker tool works correctly when interacting with the diagram.

If the issue persists, please provide additional details about the event handling logic or any other relevant parts of the codebase.
