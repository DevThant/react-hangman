Given the additional context and the provided code for the `eventService` and `useFocusable`, the issue likely stems from how the picker tool interacts with the scroll events. Here's a structured approach to diagnose and resolve the issue:

### Analysis and Potential Issues
1. **Event Handling Interference**: The picker tool might be intercepting scroll events, preventing them from propagating correctly.
2. **Focus Management**: The `useFocusable` composable could be affecting how focus and blur events are managed, potentially interfering with scrolling behavior.
3. **Event Propagation**: The `eventService` might be emitting events that change the behavior of components when the picker tool is active.

### Step-by-Step Diagnosis and Fixes

#### Step 1: Ensure Event Propagation
Ensure that the scroll events are correctly propagated when the picker tool is enabled. Check if any event listener or handler might be preventing this.

#### Example: Modify Picker Tool Event Handling
Add logging to ensure that scroll events are not being intercepted by the picker tool:

```typescript
const onScrollEvent = (event: Event) => {
  console.log('Scroll event:', event);
  // Ensure event propagation is not stopped
  event.stopPropagation = false;
  event.preventDefault = false;
};

const onDiagramMounted = (target: string) => {
  gojsDiagram.attachDiagram(target);
  // Attach scroll event listener to the diagram
  gojsDiagram.diagram.addDiagramListener('DocumentScroll', onScrollEvent);
  console.log('Diagram mounted:', target);
};
```

#### Step 2: Inspect and Adjust Focus Management
Ensure that the `useFocusable` composable is not interfering with the scroll events. Ensure that the focus and blur methods do not interfere with the scroll behavior.

#### Example: Focus Management
```typescript
function focus(): void {
  if (focusElement?.value?.focus) {
    focusElement.value.focus();
  } else if (focusElement?.value?.mainElement?.focus) {
    focusElement.value.mainElement.focus();
  } else if ((focusElement?.value as any)?.$el?.focus) {
    (focusElement.value as any).$el.focus();
  }
  console.log('Element focused:', focusElement.value);
}

function blur(): void {
  if (focusElement?.value?.blur) {
    focusElement.value.blur();
  } else if (focusElement?.value?.mainElement?.blur) {
    focusElement.value.mainElement.blur();
  } else if ((focusElement?.value as any)?.$el?.blur) {
    (focusElement.value as any).$el.blur();
  }
  console.log('Element blurred:', focusElement.value);
}
```

### Step 3: Modify Scroll Handling in Diagram and Side Panel
Ensure that the diagram and side panel handle scroll events correctly and do not interfere with each other.

#### Example: Modify Scroll Handling
```typescript
const onDiagramMounted = (target: string) => {
  gojsDiagram.attachDiagram(target);
  // Prevent the diagram from intercepting scroll events when the picker is active
  gojsDiagram.diagram.addDiagramListener('DocumentScroll', (event) => {
    if (isPickerActive()) {
      console.log('Picker active, scroll event:', event);
      event.bubbles = true;
      event.handled = false;
    }
  });
};

const isPickerActive = () => {
  return diagramStore.selectedToolItem?.toolType === ToolType.PickerTool;
};
```

### Step 4: Debug Event Emission
Check the events emitted by `eventService` and ensure they are not causing the scroll behavior to change unexpectedly.

#### Example: Debug Event Emission
```typescript
eventService.on(EventType.Shortcut, (shortcut) => {
  console.log('Shortcut event:', shortcut);
});

eventService.on(EventType.Click, (event) => {
  console.log('Click event:', event);
});
```

### Conclusion
By following these steps, you can diagnose and potentially resolve the issue with scroll events being intercepted or improperly handled when the picker tool is active. If these steps do not resolve the issue, consider providing more specific parts of the codebase related to event handling or scroll management for further analysis.
