If your project has ESLint enabled and does not allow `console.log`, it's common to use a logging library or a custom logging service. From the code snippets you provided, it looks like there is a `loggingService` available in the project.

Let's use `loggingService` for logging in the modifications:

### Detailed Investigation Steps with loggingService

#### 1. **Check Scroll Handling in the Picker Tool**
The picker tool might be intercepting scroll events or causing focus changes that interfere with normal scroll behavior.

**TrackItemProperty.vue**
Add logging to the `toggleEdit` method to see if it interferes with scroll events.

```typescript
const toggleEdit = (): void => {
  if (diagramStore.selectedToolItem?.category !== props.category) {
    diagramStore.selectToolItem({
      category: props.category as ToolItemCategory,
      toolType: ToolType.PickerTool,
      toolName: 'MultiReferencePickerTool',
      id: props.id
    });

    showAll.value = true;
    if (!isAllValuesHighlighted()) {
      highlight(props.value);
    }
    loggingService.info('Picker tool activated:', { toolItem: diagramStore.selectedToolItem });
    return;
  }

  diagramStore.selectToolItem(null);
  loggingService.info('Picker tool deactivated');
};
```

#### 2. **Ensure Proper Event Propagation**
Make sure that scroll events are not being stopped by any event listeners.

**Editor.vue**
Add a listener to log scroll events and ensure they are propagated.

```typescript
const onScrollEvent = (event: Event) => {
  loggingService.info('Scroll event:', { event });
  event.stopPropagation = false;
  event.preventDefault = false;
};

const onDiagramMounted = (target: string) => {
  gojsDiagram.attachDiagram(target);
  gojsDiagram.diagram.addDiagramListener('DocumentScroll', onScrollEvent);
  loggingService.info('Diagram mounted:', { target });
};
```

#### 3. **Focus Management in useFocusable**
Check if the focus management interferes with scrolling. 

**useFocusable.ts**
Add logging to focus and blur methods.

```typescript
function focus(): void {
  if (focusElement?.value?.focus) {
    focusElement.value.focus();
  } else if (focusElement?.value?.mainElement?.focus) {
    focusElement.value.mainElement.focus();
  } else if ((focusElement?.value as any)?.$el?.focus) {
    (focusElement.value as any).$el.focus();
  }
  loggingService.info('Element focused:', { element: focusElement.value });
}

function blur(): void {
  if (focusElement?.value?.blur) {
    focusElement.value.blur();
  } else if (focusElement?.value?.mainElement?.blur) {
    focusElement.value.mainElement.blur();
  } else if ((focusElement?.value as any)?.$el?.blur) {
    (focusElement.value as any).$el.blur();
  }
  loggingService.info('Element blurred:', { element: focusElement.value });
}
```

#### 4. **Inspect eventService for Interference**
Check if the events emitted by `eventService` are causing unexpected behavior.

**eventService.js**
Add logging to see when events are emitted.

```typescript
eventService.on(EventType.Shortcut, (shortcut) => {
  loggingService.info('Shortcut event:', { shortcut });
});

eventService.on(EventType.Click, (event) => {
  loggingService.info('Click event:', { event });
});
```

### Conclusion
These steps should help identify where the interference is happening. By adding logging using `loggingService` and ensuring proper event propagation, we can narrow down the issue. If these steps do not resolve the problem, consider focusing on any custom scroll handling logic within the picker tool or related components.

Would you like to proceed with these modifications and check the logs for further analysis? Or do you have specific areas in the codebase where you suspect the problem might be?
