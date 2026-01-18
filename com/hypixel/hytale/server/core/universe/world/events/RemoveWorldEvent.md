---
description: Architectural reference for RemoveWorldEvent
---

# RemoveWorldEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Transient

## Definition
```java
// Signature
public class RemoveWorldEvent extends WorldEvent implements ICancellable {
```

## Architecture & Concepts
The RemoveWorldEvent is a transient, informational object that signals the server's intent to unload a World instance from memory. It is a critical component of the server's event-driven architecture, acting as a notification that allows various systems to react to, and potentially veto, the removal of a world.

This event does not perform the removal itself; rather, it is dispatched by the world management system *before* the removal process begins. By implementing the ICancellable interface, it establishes a contract where listeners can prevent the action from completing. This pattern provides a powerful extension point for plugins or core game logic to ensure graceful shutdowns, such as completing player data serialization or finishing a world-saving operation before the world object is destroyed.

A key architectural feature is the RemovalReason enum. The EXCEPTIONAL reason serves as a non-cancellable override, ensuring that critical cleanup operations, such as handling a world corruption or a forced server shutdown, cannot be blocked by other systems. This provides a necessary failsafe for maintaining server stability.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's high-level world management service (e.g., a UniverseManager) when it initiates the process to unload a world. This can be triggered by administrative commands, automated cleanup of unused worlds, or in response to a critical error.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus. It is a fire-and-forget data carrier.
- **Destruction:** The object becomes eligible for garbage collection immediately after all registered event listeners have processed it. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Mutable. The primary state, the boolean *cancelled* flag, is designed to be modified by one or more event listeners during the dispatch cycle. The initial state (World and RemovalReason) is immutable after construction.
- **Thread Safety:** This class is **not thread-safe** and must not be treated as such. Event dispatching in the Hytale server is expected to occur on a single, main server thread. Listeners execute sequentially. Modifying the event's state from an asynchronous task or a different thread will lead to race conditions and undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRemovalReason() | RemovalReason | O(1) | Returns the reason for the removal, either GENERAL or EXCEPTIONAL. |
| isCancelled() | boolean | O(1) | Returns true if the event has been cancelled. **Warning:** This will always return false if the RemovalReason is EXCEPTIONAL, regardless of calls to setCancelled. |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state of the event. This is the primary interaction point for event listeners. |

## Integration Patterns

### Standard Usage
The intended use is within an event listener method. The listener inspects the event's context and, if necessary, cancels it to prevent the world from being removed.

```java
// A system that prevents a world from being removed if a backup is in progress.
@Subscribe
public void onWorldRemovalAttempt(RemoveWorldEvent event) {
    World world = event.getWorld();
    BackupManager backupManager = ServiceRegistry.get(BackupManager.class);

    // Only intervene for general-purpose removals.
    if (event.getRemovalReason() == RemovalReason.GENERAL) {
        if (backupManager.isBackupInProgress(world.getId())) {
            // Veto the removal to allow the backup to complete safely.
            event.setCancelled(true);
            log.info("Delayed removal of world " + world.getName() + " due to ongoing backup.");
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of RemoveWorldEvent directly with *new*. Firing a custom instance into the event bus is not a supported mechanism for unloading a world and will have no effect, as the world management system is the sole authority that acts upon the event's final state.
- **Ignoring RemovalReason:** Logic that attempts to cancel an event must first check the RemovalReason. Attempting to cancel an EXCEPTIONAL removal is a logical error, as the system is designed to ignore the cancellation request in this scenario to guarantee cleanup.
- **Asynchronous Modification:** Do not store a reference to the event object and attempt to modify it from another thread after the listener method has returned. The event's lifecycle is tied to the synchronous dispatch call stack.

## Data Pipeline
The event acts as a control signal in the world unloading pipeline. The flow enables a clear, interruptible process for managing server resources.

> Flow:
> World Management System (Intent to Unload) -> `new RemoveWorldEvent(...)` -> Event Bus Dispatch -> Synchronous Execution of Listeners (Potential call to `setCancelled`) -> World Management System (Checks `isCancelled()`) -> **If Not Cancelled:** Proceed with World Unload & Resource Cleanup / **If Cancelled:** Abort Unload Process

