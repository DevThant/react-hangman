Let's address the issues you encountered and make the necessary corrections to ensure everything works as expected. Here is the revised plan and implementation.

### Step-by-Step Implementation Plan

1. **Add Sync Progress Event in event.ts:**
   - Add a new event type `SyncProgress` to the `EventType` enum.
   - Add the event in the event service.

2. **Update Sync.vue to Emit Sync Progress:**
   - Ensure that the `Sync.vue` component emits the progress of the sync process.
   - Create an event that updates the status of the sync progress.

3. **Update Spinner.vue to Listen for Sync Progress:**
   - Modify `Spinner.vue` to listen for the sync progress event.
   - Update the displayed message to include the number of models synced.

4. **Modify Toaster Service:**
   - Ensure the toaster service can handle the new type of toast message with progress information.

### Implementation

#### 1. Add Sync Progress Event in `event.ts`

```typescript
import mitt, { Emitter, Handler } from "mitt";

import { Shortcut as ShortcutTypes } from "@/typings/shortcut.js";
import { DialogOptions, DialogNames } from "@/typings/dialog.js";
import { Key } from "@/typings/keyboard.js";
import { Panel } from "@/typings/sidePanel.js";

export interface ErrorOptions {
  retryable?: boolean;
  titleTranslationKey?: string;
  bodyTranslationKey?: string;
  error?: Error;
}

export enum EventType {
  ScaleToFit = "scaleToFit",
  CenterDiagram = "centerDiagram",
  Shortcut = "shortcut",
  Click = "click",
  OpenDialog = "openDialog",
  CloseDialog = "closeDialog",
  Movement = "movement",
  CenterSelection = "centerSelection",
  CenterHighlighted = "centerHighlighted",
  ResetSearch = "resetSearch",
  GoToErrorPage = "goToErrorPage",
  DomainObjectUpdated = "domainObjectUpdated",
  DomainObjectDeleted = "domainObjectDeleted",
  ToggleSidePanel = "toggleSidePanel",
  ToggleToolsPanel = "toggleToolsPanel",
  OpenToolsPanel = "openToolsPanel",
  GenerateSVG = "GenerateSVG",
  CosNodePlaced = "CosNodePlaced",
  NotifyDetailsTable = "NotifyDetailsTable",
  CenterIfOutOfView = "CenterIfOutOfView",
  SyncProgress = "syncProgress" // Add this line
}

type Events = {
  [EventType.ScaleToFit]: undefined;
  [EventType.CenterDiagram]: undefined;
  [EventType.Shortcut]: ShortcutTypes;
  [EventType.Click]: MouseEvent;
  [EventType.OpenDialog]: { dialogName: DialogNames; options?: DialogOptions };
  [EventType.CloseDialog]: undefined;
  [EventType.Movement]: Key;
  [EventType.CenterSelection]: undefined;
  [EventType.CenterHighlighted]: undefined;
  [EventType.ResetSearch]: undefined;
  [EventType.GoToErrorPage]: ErrorOptions;
  [EventType.DomainObjectUpdated]: string;
  [EventType.NotifyDetailsTable]: string;
  [EventType.DomainObjectDeleted]: string;
  [EventType.ToggleSidePanel]: { panel: Panel; options?: Record<string, any> };
  [EventType.ToggleToolsPanel]: undefined;
  [EventType.OpenToolsPanel]: undefined;
  [EventType.GenerateSVG]: undefined;
  [EventType.CosNodePlaced]: undefined;
  [EventType.CenterIfOutOfView]: undefined;
  [EventType.SyncProgress]: { syncedModels: number; totalModels: number }; // Add this line
};

/**
 * Implements a global event bus where events can be fired and listened too anywhere in the app.
 */
function createEventService() {
  const { emit, on, off }: Emitter<Events> = mitt<Events>();

  function once<T extends keyof Events>(
    event: T,
    callback: Handler<Events[T]>
  ): void {
    const fn = (args: Events[T]) => {
      callback(args);
      off(event, fn);
    };

    on(event, fn);
  }

  return {
    on,
    off,
    emit,
    once,
  };
}

export const eventService = createEventService();
```

#### 2. Update `Sync.vue` to Emit Sync Progress

```vue
<template>
  <!-- Existing template code... -->
</template>

<script setup lang="ts">
import { ref, watch, computed } from "vue";
import { eventService, EventType } from "@/services/event.js";
// Existing imports...

const status = ref(props.status || { syncedModels: [], allModels: [] });

watch(
  () => status.value,
  (newStatus) => {
    eventService.emit(EventType.SyncProgress, {
      syncedModels: newStatus.syncedModels.length,
      totalModels: newStatus.allModels.length,
    });
  },
  { deep: true }
);

// Existing script code...
</script>

<style scoped>
/* Existing style code... */
</style>
```

#### 3. Update `Spinner.vue` to Listen for Sync Progress

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
    <div v-if="progressMessage" class="progress-message">{{ progressMessage }}</div>
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from "vue";
import { toastService, ToastEvent } from "@/services/toast.js";
// Existing imports...

const progressMessage = ref("");

const updateProgress = (event: { syncedModels: number; totalModels: number }) => {
  progressMessage.value = `${event.syncedModels}/${event.totalModels} models are done.`;
};

onMounted(() => {
  toastService.on(ToastEvent.SyncProgress, updateProgress);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.SyncProgress, updateProgress);
});

// Existing script code...
</script>

<style scoped>
/* Existing style code... */
.progress-message {
  margin-bottom: 10px;
}
</style>
```

#### 4. Modify `toast.ts` to Handle Sync Progress Events

```typescript
import { DefineComponent } from "vue";
import mitt, { Emitter, Handler } from "mitt";
import { LoggingService } from "@ebitoolmx/logging-service/console";
import { loggingService as commonLoggingService } from "./logging.js";

export enum ToastEvent {
  Display = "display",
  Dismiss = "dismiss",
  CloseAll = "closeAll",
  SyncProgress = "syncProgress" // Add this line
}

export enum ToastType {
  Simple = "simple",
  UpdateAvailable = "updateAvailable",
  Spinner = "spinner",
  Reconnecting = "reconnecting",
  PickUpWhereYouLeftOff = "pickUpWhereYouLeftOff",
}

/**
 * All the required details for displaying a toast in the Toaster component.
 */
export interface Toast {
  id: symbol;
  component?: DefineComponent;
  properties: Record<string, unknown>;
  timeout?: number;
}

type Events = {
  [ToastEvent.Display]: Toast;
  [ToastEvent.Dismiss]: symbol;
  [ToastEvent.CloseAll]: undefined;
  [ToastEvent.SyncProgress]: { syncedModels: number; totalModels: number }; // Add this line
};

/**
 * Handles toaster events and components for the toaster component.
 */
export class ToastService {
  protected toastComponents = new Map<ToastType, DefineComponent>();

  protected events: Emitter<Events> = mitt<Events>();

  public constructor(protected loggingService: LoggingService) {}

  /**
   * Registers a toast into the service so that it can be referenced later when displaying
   */
  public registerToast(type: ToastType, toast: any) {
    this.toastComponents.set(type, toast);
  }

  /**
   * Displays a toast with a given id and of specified type. This will emit an event to the Toaster
   * component with the necessary details to display a toast. This is the most low-level endpoint
   * that all details to be able to display a toast.
   */
  public displayWithId(
    id: symbol,
    type: ToastType,
    props: Record<string, unknown> = {},
    timeout: number | undefined = undefined
  ): symbol {
    const properties = props;
    const component = this.toastComponents.get(type);

    if (!component)
      this.loggingService.error(
        `Toast type '${type}' has not been registered.`
      );

    this.events.emit(ToastEvent.Display, {
      id,
      component,
      properties,
      timeout,
    });
    this.loggingService.debug(`Displaying toast of type '${type}'`);

    return id;
  }

  /**
   * Displays a toast of a specified type. This will auto generate an id and return it before
   * emitting an event to the Toaster component. See displayWithId().
   */
  public display(
    type: ToastType,
    props: Record<string, unknown> = {},
    timeout: number | undefined = undefined
  ): symbol {
    return this.displayWithId(Symbol("toast"), type, props, timeout);
  }

  /**
   * Displays a simple info toast with a provided message and optional timeout
   */
  public displayInfo(message: string, timeout?: number): symbol;

  /**
   * Displays a simple info toast with a provided header, message, and optional timeout
   */
  public displayInfo(header

: string, message: string, timeout?: number): symbol;

  /**
   * A fallback implementation for displayInfo that will determine which argument is the toast
   * message, header, and timeout based on the argument types.
   */
  public displayInfo(
    arg1: string,
    arg2?: string | number,
    arg3?: number
  ): symbol {
    const header: string | undefined =
      typeof arg2 === "string" ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === "string" ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === "number" ? arg2 : arg3;

    return this.display(
      ToastType.Simple,
      { title: header, text: message, type: "info" },
      timeout
    );
  }

  /**
   * Displays a simple warning toast with a provided message and optional timeout
   */
  public displayWarning(message: string, timeout?: number): symbol;

  /**
   * Displays a simple warning toast with a provided header, message, and optional timeout
   */
  public displayWarning(
    header: string,
    message: string,
    timeout?: number
  ): symbol;

  /**
   * A fallback implementation for displayWarning that will determine which argument is the toast
   * message, header, and timeout based on the argument types.
   */
  public displayWarning(
    arg1: string,
    arg2?: string | number,
    arg3?: number
  ): symbol {
    const header: string | undefined =
      typeof arg2 === "string" ? arg1 : undefined;
    const message: string | undefined = typeof arg2 === "string" ? arg2 : arg1;
    const timeout: number | undefined = typeof arg2 === "number" ? arg2 : arg3;

    return this.display(
      ToastType.Simple,
      { title: header, text: message, type: "warning" },
      timeout
    );
  }

  /**
   * Displays a simple error toast with a provided message
   */
  public displayError(message: string): symbol;

  /**
   * Displays a simple error toast with a provided message and header
   */
  public displayError(header: string, message: string): symbol;

  /**
   * A fallback implementation for typing.
   */
  public displayError(arg1: string, arg2?: string): symbol {
    const header = arg2 ? arg1 : undefined;
    const message = arg2 ?? arg1;

    return this.display(ToastType.Simple, {
      title: header,
      text: message,
      type: "error",
    });
  }

  /**
   * Will display a spinner toaster with the provided id and header.
   */
  public displaySpinner(id: symbol, header: string): symbol {
    return this.displayWithId(id, ToastType.Spinner, { title: header });
  }

  /**
   * Dismisses a toast with the provided id if it is still present on the screen.
   */
  public dismiss(id: symbol): void {
    this.events.emit(ToastEvent.Dismiss, id);
  }

  /**
   * Closes all toasts
   */
  public closeAll(): void {
    this.events.emit(ToastEvent.CloseAll);
  }

  /**
   * Subscribe to the closeAll toast event in the toast service
   */
  public on<Type extends ToastEvent.CloseAll>(
    type: Type,
    callback: Function
  ): void;

  /**
   * Subscribe to the dismiss toast event in the toast service
   */
  public on<Type extends ToastEvent.Dismiss>(
    type: Type,
    callback: Function
  ): void;

  /**
   * Subscribe to the display toast event. This event will pass information about the toast to be
   * displayed.
   */
  public on<Type extends ToastEvent.Display>(
    type: Type,
    callback: (toast: Toast) => void
  ): void;

  /**
   * Subscribe to the sync progress event in the toast service.
   */
  public on<Type extends ToastEvent.SyncProgress>(
    type: Type,
    callback: (progress: { syncedModels: number; totalModels: number }) => void
  ): void;

  /**
   * A fallback type for subscribing to events in the toast service.
   */
  public on<T extends ToastEvent>(type: T, callback: Handler<Events[T]>): void {
    this.events.on(type, callback);
  }

  /**
   * Unsubscribe from from an event in the toast service
   */
  public off<T extends ToastEvent>(
    type: T,
    callback: Handler<Events[T]>
  ): void {
    this.events.off(type, callback);
  }
}

export const toastService = new ToastService(commonLoggingService);
```

This code ensures that the `Sync.vue` component emits progress updates via the `eventService`, and `Spinner.vue` listens for these updates and reacts accordingly.

Relevant Hotkeys:
- **W:** Yes, Continue - Confirm, advance to next step, proceed, again
- **S:** Explain - Explain each line of code step by step, adding descriptive comments
- **D:** Iterate, Improve, Evolve - Note 3 critiques or edge cases, propose improvements 1,2,3
- **G:** Stash sandbox - Write files data mnt
- 
-----------

Greetings Traveler,
Grim-terface v2.5 🧙‍♂️ delved

Let’s begin our coding quest!

### Step-by-Step Implementation Plan

1. **Update Sync.vue to Emit Sync Progress:**
   - Ensure that the `Sync.vue` component emits the progress of the sync process.
   - Create an event that updates the status of the sync progress.

2. **Update Spinner.vue to Listen for Sync Progress:**
   - Modify `Spinner.vue` to listen for the sync progress event.
   - Update the displayed message to include the number of models synced.

3. **Modify Toaster Service:**
   - Ensure the toaster service can handle the new type of toast message with progress information.
   
4. **Ensure Real-time Updates:**
   - Use reactive data binding to ensure the spinner toast updates in real-time as the sync progresses.

### Implementation

#### 1. Update `Sync.vue` to Emit Sync Progress

First, update `Sync.vue` to emit progress events during the sync process:

```vue
<template>
  <!-- Existing template code... -->
</template>

<script setup lang="ts">
import { ref, watch, computed } from "vue";
// Existing imports...

const status = ref(props.status || { syncedModels: [], allModels: [] });

watch(
  () => status.value,
  (newStatus) => {
    eventService.emit(EventType.SyncProgress, {
      syncedModels: newStatus.syncedModels.length,
      totalModels: newStatus.allModels.length,
    });
  },
  { deep: true }
);

// Existing script code...
</script>

<style scoped>
/* Existing style code... */
</style>
```

#### 2. Update `Spinner.vue` to Listen for Sync Progress

Modify `Spinner.vue` to listen for sync progress events and update its message accordingly:

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
    <div v-if="progressMessage" class="progress-message">{{ progressMessage }}</div>
    <app-horizontal-spinner class="spinner" :thin="true"></app-horizontal-spinner>
  </div>
</template>

<script setup lang="ts">
import { ref, onBeforeUnmount, onMounted } from "vue";
// Existing imports...

const progressMessage = ref("");

const updateProgress = (event: { syncedModels: number, totalModels: number }) => {
  progressMessage.value = `${event.syncedModels}/${event.totalModels} models are done.`;
};

onMounted(() => {
  toastService.on(ToastEvent.SyncProgress, updateProgress);
});

onBeforeUnmount(() => {
  toastService.off(ToastEvent.SyncProgress, updateProgress);
});

// Existing script code...
</script>

<style scoped>
/* Existing style code... */
.progress-message {
  margin-bottom: 10px;
}
</style>
```

#### 3. Modify `toast.ts` to Handle Sync Progress Events

Ensure the toast service can handle sync progress events:

```typescript
// Existing imports...
export enum ToastEvent {
  Display = "display",
  Dismiss = "dismiss",
  CloseAll = "closeAll",
  SyncProgress = "syncProgress", // Add this line
}

// Existing ToastService class...

export class ToastService {
  // Existing methods...

  public updateSyncProgress(syncedModels: number, totalModels: number) {
    this.events.emit(ToastEvent.SyncProgress, { syncedModels, totalModels });
  }

  // Existing methods...
}

export const toastService = new ToastService(commonLoggingService);
```

#### 4. Ensure Real-time Updates

The `Sync.vue` component already emits progress updates via the `eventService`, and `Spinner.vue` listens for these updates and reacts accordingly.

### Deploy Updated Code

The code provided ensures the spinner toast in `Spinner.vue` updates in real-time with the sync progress.



Relevant Hotkeys:
- **W:** Yes, Continue - Confirm, advance to next step, proceed, again
- **S:** Explain - Explain each line of code step by step, adding descriptive comments
- **D:** Iterate, Improve, Evolve - Note 3 critiques or edge cases, propose improvements 1,2,3
- **G:** Stash sandbox - Write files data mnt
