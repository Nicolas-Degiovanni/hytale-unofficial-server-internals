---
description: Architectural reference for MaterialContainerWindow
---

# MaterialContainerWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Interface

## Definition
```java
// Signature
public interface MaterialContainerWindow {
```

## Architecture & Concepts
The MaterialContainerWindow interface defines a strict contract for server-side UI window implementations that manage item materials or special resources. It serves as a standardized abstraction layer, allowing core systems like inventory management and crafting logic to interact with any compatible window without needing to know its specific concrete type (e.g., a crafting table, a furnace, or a custom modded block).

This interface is a key component of the server's inventory system. It decouples the logic for validating and updating resource-dependent UI sections from the window's primary function. By enforcing this contract, the system ensures that any window presenting complex material requirements can be correctly invalidated and refreshed when underlying player inventory or game state changes.

## Lifecycle & Ownership
As an interface, MaterialContainerWindow does not have its own lifecycle. Instead, it defines the expected behavior for implementing classes, whose lifecycle is dictated by player interaction.

- **Creation:** A concrete object implementing this interface is typically instantiated by the server when a player interacts with a specific world object (e.g., a workbench block) that opens a corresponding UI.
- **Scope:** The implementing object's lifetime is tied directly to the player's UI session with that object. It exists only as long as the window is considered "open" for that player.
- **Destruction:** The object is marked for garbage collection when the player closes the window or is disconnected from the server. The `isValid` method is the primary mechanism for external systems to check if the underlying window is still active.

## Internal State & Concurrency
The interface itself is stateless, but it implies that any implementing class will manage significant internal state.

- **State:** Implementations are expected to be highly mutable. They must track the state of their material and resource slots. The `invalidateExtraResources` method explicitly signals a requirement for the implementation to re-evaluate and potentially modify its internal state based on external changes.
- **Thread Safety:** **WARNING:** All implementations of this interface must be thread-safe. Window objects are part of the player's state on the server and can be accessed by the main server tick thread as well as network threads processing player input. Implementations must use appropriate synchronization mechanisms to prevent race conditions when updating or querying window state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getExtraResourcesSection() | MaterialExtraResourcesSection | O(1) | Returns the component responsible for managing the specialized material or resource slots of this window. |
| invalidateExtraResources() | void | O(N) | Triggers a re-computation of the extra resources section. Complexity depends on the implementation's logic. |
| isValid() | boolean | O(1) | Checks if the window is still active and valid for interaction. Returns false if the player has closed the UI. |

## Integration Patterns

### Standard Usage
Systems should retrieve the player's current window and use an `instanceof` check to safely cast to MaterialContainerWindow before interacting with it. This pattern ensures that the logic only runs for windows that explicitly support material management.

```java
// How a system should interact with a player's open window
PlayerWindow window = player.getCurrentWindow();

if (window instanceof MaterialContainerWindow && window.isValid()) {
    MaterialContainerWindow materialWindow = (MaterialContainerWindow) window;
    materialWindow.invalidateExtraResources();
}
```

### Anti-Patterns (Do NOT do this)
- **Stale References:** Do not cache a reference to a MaterialContainerWindow object. Always re-fetch the player's current window and check `isValid()` before use, as the player may have closed it.
- **Unchecked Casting:** Never assume a PlayerWindow is a MaterialContainerWindow. Failure to perform an `instanceof` check will lead to ClassCastExceptions.
- **Ignoring Validity:** Calling any method on a window where `isValid()` returns false is undefined behavior and may lead to server instability or exceptions.

## Data Pipeline
This interface acts as a control point within the server-side UI data flow. It allows external systems to trigger state changes within a window implementation.

> Flow:
> Player Inventory Change -> InventoryManager Event -> **MaterialContainerWindow.invalidateExtraResources()** -> Concrete Window recalculates state -> Server sends network packet to client -> Client UI updates visually

