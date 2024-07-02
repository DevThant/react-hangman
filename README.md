### SplitView
```typescript
const drag = (event: MouseEvent) => {
  const splitViewBounds = splitView.value?.getBoundingClientRect();
  const tabBounds = tabs.value?.getBoundingClientRect();

  const mousePosY = event.clientY;

  let splitViewTargetHeight: number;
  if (splitViewBounds && tabBounds) {
    const offsetMouseYPos = mousePosY - tabBounds.height - splitViewBounds.top;
    splitViewTargetHeight = (offsetMouseYPos / splitViewBounds.height) * 100;
  } else {
    const viewportHeight = window.innerHeight;
    splitViewTargetHeight = (mousePosY / viewportHeight) * 100;
  }

  const boundSplitViewHeight = Math.min(Math.max(splitViewTargetHeight, 10), 90);
  const formattedSplitViewHeight = `${boundSplitViewHeight.toFixed(3)}%`;

  const diagram = document.getElementById('diagram');
  if (!diagram) return;

  // Ensure this does not interfere with SidePanel events
  if (event.target && (event.target as HTMLElement).closest('.side-panel')) {
    return;
  }

  diagram.style.height = formattedSplitViewHeight;
  preferencesStore.initialSplitViewSize = formattedSplitViewHeight;
};


...

onMounted(() => {
  const diagram = document.getElementById('diagram');
  if (!diagram) return;
  diagram.style.height = preferencesStore.initialSplitViewSize;

  // Focus management
  diagram.addEventListener('focus', (event) => {
    const target = event.target as HTMLElement;
    if (target && target.closest('.side-panel')) {
      event.stopPropagation();
    }
  }, true);
});

```
### PickerTool
```typescript
 doMouseDown(event: MouseEvent) {
  super.doMouseDown(event);
  // Ensure event.target is not null and cast it to HTMLElement to use closest
  if (event.target && (event.target as HTMLElement).closest('.side-panel')) {
    event.stopPropagation();
  }
  this.eventBus.emit('mxClickToolActivated');
}

doMouseUp(event: MouseEvent) {
  super.doMouseUp(event);
  // Ensure event.target is not null and cast it to HTMLElement to use closest
  if (event.target && (event.target as HTMLElement).closest('.side-panel')) {
    event.stopPropagation();
  }
  this.eventBus.emit('mxClickToolDeactivated');
}  
```
-----
Summary of Potential Issues and Solutions:
Event Propagation:

Ensure that the events (mouse events, especially mousedown, mousemove, and mouseup) are correctly propagated between the SplitView and SidePanel components.
Check if the drag method in SplitView interferes with events in SidePanel.
Focus Management:

Verify that the focus is managed correctly when interacting with the diagram and the side panel. Ensure that activating the picker tool does not force the focus to the diagram, which can cause the side panel to lose focus.
Event Handling in SidePanel:

The SidePanel component relies on computed properties and event services to manage visibility and interaction. Ensure these events are correctly handled and do not interfere with the split view's resize or scroll events.
Recommended Fixes:
Allow Event Propagation:

Modify event handlers in SplitView to ensure they do not prevent default behavior unnecessarily.
```typescript
const drag = (event: MouseEvent) => {
  // Existing logic

  // Ensure this does not interfere with SidePanel events
  if (event.target.closest('.side-panel')) {
    return;
  }

  // Existing logic
};
```
Focus Management:

Ensure the focus remains on the side panel or the active input element within it.
```typescript
onMounted(() => {
  const diagram = document.getElementById('diagram');
  if (!diagram) return;
  diagram.style.height = preferencesStore.initialSplitViewSize;

  // Focus management
  diagram.addEventListener('focus', (event) => {
    if (event.target.closest('.side-panel')) {
      event.stopPropagation();
    }
  }, true);
});
```
Check Event Handling in Picker Tool:

Modify the doMouseDown and doMouseUp methods in the picker tool to ensure events are propagated correctly.
```typescript
\
doMouseDown(event) {
  super.doMouseDown(event);
  if (event.target.closest('.side-panel')) {
    event.stopPropagation();
  }
  this.eventBus.emit('mxClickToolActivated');
}

doMouseUp(event) {
  super.doMouseUp(event);
  if (event.target.closest('.side-panel')) {
    event.stopPropagation();
  }
  this.eventBus.emit('mxClickToolDeactivated');
}
```
Testing:
Test Scroll and Input in SidePanel:
After making the changes, test the scroll and input functionalities in the SidePanel when the picker tool is activated.
Monitor Event Propagation:
Use browser developer tools to monitor event propagation and ensure that no events are blocked or improperly handled.
By making these adjustments, you should be able to resolve the scroll and input issues in the SidePanel when the picker tool is activated. If further assistance is needed, please provide specific parts for deeper investigation.
