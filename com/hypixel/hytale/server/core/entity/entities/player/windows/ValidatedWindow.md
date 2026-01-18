---
description: Architectural reference for the ValidatedWindow interface.
---

# ValidatedWindow

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface ValidatedWindow {
```

## Architecture & Concepts
The ValidatedWindow interface defines a strict behavioral contract for any server-side window or container that requires state validation. It serves as a critical component in the server's authority model, ensuring that the state of a player's open inventory or UI window remains consistent and synchronized with the server's game logic.

This contract is primarily enforced during player interactions. When a player performs an action (e.g., moving an item, clicking a button), the server must verify that the action is still valid in the current game state. For example, a crafting window must validate that the player still possesses the required ingredients before executing the craft. By mandating a validation method, this interface decouples the validation logic from the core window management system, allowing for diverse and complex window implementations.

## Lifecycle & Ownership
As an interface, ValidatedWindow itself does not have a lifecycle. Instead, it imposes lifecycle-related responsibilities on its implementing classes.

- **Creation:** *Implementing objects* are typically instantiated when a player interacts with a game object that opens a window, such as a chest or a crafting table. This is managed by a factory or manager class within the windowing system.
- **Scope:** The lifespan of an *implementing object* is tied directly to the player's interaction. It persists as long as the player has the corresponding window open.
- **Destruction:** The object is marked for garbage collection when the player closes the window or is disconnected from the server. The WindowManager is responsible for this cleanup.

## Internal State & Concurrency
The interface defines no state. However, classes that implement ValidatedWindow are expected to be stateful.

- **State:** *Implementations* will hold mutable state representing the contents and properties of the window (e.g., item stacks, slot configurations). This state is the primary target of the validation logic.
- **Thread Safety:** **WARNING:** Implementations of this interface are not expected to be thread-safe. All interactions, including calls to the validate method, must be performed on the main server thread to prevent race conditions and data corruption related to inventory and world state.

## API Surface
The public contract consists of a single method designed to be a gatekeeper for state-changing operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate() | boolean | O(N) | Checks the internal consistency of the window. Returns true if the state is valid, false otherwise. Complexity is dependent on the implementation, often proportional to the number of slots or elements in the window. |

## Integration Patterns

### Standard Usage
The primary integration pattern involves a managing system invoking the validate method before processing any player actions related to the window. This ensures that no operation proceeds based on a stale or invalid state.

```java
// A WindowManager or similar system checks validity before processing a packet.
ValidatedWindow currentWindow = player.getOpenWindow();

if (!currentWindow.validate()) {
    // Force-close the window or send a resync packet to the client.
    player.closeWindow(WindowCloseReason.STATE_INVALID);
    return;
}

// Proceed with processing the player's action...
```

### Anti-Patterns (Do NOT do this)
- **Empty Implementation:** Implementing this interface but providing an empty or perpetually true-returning validate method completely undermines server authority and can lead to item duplication or loss.
- **Client-Side Trust:** The validate method must *never* trust data sent from the client without re-validating it against the server's current state.
- **Ignoring Failure:** Calling validate but failing to handle a false return value. A failed validation is a critical error that indicates a state desynchronization between client and server, which must be corrected immediately.

## Data Pipeline
This interface acts as a validation gate within the server's data processing pipeline for player interactions.

> Flow:
> Client Input Packet -> Packet Handler -> **ValidatedWindow.validate()** -> [If true] Game State Mutation -> [If false] State Correction / Disconnect

