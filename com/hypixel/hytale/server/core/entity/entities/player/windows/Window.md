---
description: Architectural reference for the Window abstract base class.
---

# Window

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Transient

## Definition
```java
// Signature
public abstract class Window {
```

## Architecture & Concepts
The Window class is the abstract, server-side representation of a user interface element that a player interacts with. It is the foundational component for all interactive UI screens, such as inventories, crafting stations, chests, and dialogue boxes.

This class is not a service but a stateful, transient object. Each instance corresponds to a single, open window for a specific player. Its primary responsibility is to manage the state and business logic of that UI interaction, acting as the authoritative model in the client-server UI architecture. It receives actions from the client via WindowAction packets, processes them, mutates its internal state, and prepares a serialized representation of that state to be synchronized back to the client.

The system is managed by a higher-level component, the WindowManager, which orchestrates the lifecycle of all Window instances for a given player.

### Lifecycle & Ownership
The lifecycle of a Window instance is tightly controlled and ephemeral, existing only for the duration of a player's interaction with a specific UI.

- **Creation:** Window instances are never created directly. Subclasses are instantiated on-demand by the WindowManager, typically in response to a player action (e.g., right-clicking a chest). The static map CLIENT_REQUESTABLE_WINDOW_TYPES functions as a factory registry, mapping a WindowType enum to a Supplier that constructs the appropriate Window subclass.
- **Scope:** An instance's scope is bound to a single player session and a single open UI. It persists only as long as the corresponding window is open on the client.
- **Destruction:** A Window is marked for destruction when its `close` method is invoked, which delegates the actual removal to the owning WindowManager. The `onClose` hook is called for final cleanup, after which the object loses all strong references from the manager and becomes eligible for garbage collection.

## Internal State & Concurrency
- **State:** The state is highly mutable. Key state includes the unique `id`, a reference to the owning `playerRef`, and the serialized data model represented by the `JsonObject` returned from `getData`. Two critical atomic flags, `isDirty` and `needRebuild`, are used to signal the need for state synchronization with the client.
- **Thread Safety:** This class is not fully thread-safe and assumes its core logic (e.g., `handleAction`, `onOpen`, `onClose`) is executed on a single, per-player game thread. However, the state-change flags (`isDirty`, `needRebuild`) are implemented using AtomicBoolean, allowing other systems on different threads to safely signal that the window's state has been externally modified and requires an update. Direct mutation of other fields from outside the game thread is an anti-pattern and will lead to race conditions.

## API Surface
The public contract is designed around lifecycle hooks, state management, and event handling.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(playerRef, manager) | void | O(1) | Initializes the window with essential context. **WARNING:** This must be called immediately after instantiation and before any other method. |
| getData() | JsonObject | O(N) | Abstract method. Subclasses must implement this to serialize their complete state into a JSON object for the client. |
| onOpen() | boolean | Varies | Lifecycle hook called by the WindowManager when the window is first opened. Returning false will cancel the open operation. |
| onClose() | void | Varies | Lifecycle hook called by the WindowManager just before the window is destroyed. Used for resource cleanup. |
| handleAction(ref, store, action) | void | Varies | The primary entry point for processing player input received from the client. |
| close() | void | O(1) | Signals the owning WindowManager to close this window. This is the correct way to programmatically close a window. |
| invalidate() | void | O(1) | Marks the window as dirty, scheduling its state to be synchronized with the client on the next update tick. |
| registerCloseEvent(consumer) | EventRegistration | O(1) | Registers a listener that will be invoked when this specific window instance is closed. |

## Integration Patterns

### Standard Usage
A developer's primary interaction is through subclassing. The subclass implements the abstract methods to define the window's specific behavior and state. The system handles instantiation and lifecycle.

```java
// In a subclass, e.g., CraftingTableWindow.java

@Override
@Nonnull
public JsonObject getData() {
    // Serialize crafting grid and output slot state to JSON
    JsonObject data = new JsonObject();
    // ... populate data
    return data;
}

@Override
public void handleAction(@Nonnull Ref<EntityStore> ref, @Nonnull Store<EntityStore> store, @Nonnull WindowAction action) {
    // Process a click on a crafting slot based on action data
    int slotId = action.getSlotId();
    // ... update internal state
    this.invalidate(); // Mark state as changed
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MyWindow()`. The WindowManager is the sole owner and creator of Window instances. Bypassing it will result in a non-functional window that is not registered with the player.
- **State Leakage:** Do not hold references to a Window instance after it has been closed. The object is considered dead and will be garbage collected. Use the `registerCloseEvent` to clean up any external listeners.
- **Ignoring `invalidate`:** Modifying the internal state of a Window without calling `invalidate` or `setNeedRebuild` will cause desynchronization. The server will have the new state, but the client will not receive an update, leading to a broken UI.
- **Pre-Init Access:** Accessing the `playerRef` or `manager` fields before the `init` method has been called by the WindowManager will result in a NullPointerException.

## Data Pipeline
The Window class is a key component in the server-to-client UI data flow. It processes incoming actions and generates outgoing state updates.

> Flow:
> Client UI Interaction -> `WindowAction` Packet -> Server Network Layer -> `WindowManager` -> **`Window.handleAction()`** -> Internal State Mutation -> **`Window.invalidate()`** -> `WindowManager` detects dirty state -> **`Window.getData()`** -> State Sync Packet -> Client UI Update

