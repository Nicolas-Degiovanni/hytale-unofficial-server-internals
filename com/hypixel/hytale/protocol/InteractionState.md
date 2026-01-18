---
description: Architectural reference for InteractionState
---

# InteractionState

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum InteractionState {
```

## Architecture & Concepts
InteractionState is a foundational enumeration within the Hytale network protocol. It serves as a type-safe, canonical representation for the outcome of a discrete player interaction, such as using an item, clicking a UI element, or interacting with a world object.

This enum acts as a contract between the client and server. When an interaction is processed, the authoritative side (typically the server) returns one of these states. This decouples the low-level network serialization from the high-level game logic. Instead of passing ambiguous "magic numbers" through the system, the protocol layer deserializes the integer value into a strongly-typed InteractionState instance. Game systems can then react to this state in a readable and robust manner, typically via a switch statement, without needing to know the underlying integer representation.

The pre-cached VALUES array is a performance optimization to prevent repeated array allocation when calling the static fromValue method, which is critical in high-frequency network code.

### Lifecycle & Ownership
- **Creation:** All instances of this enum are constants, created and managed by the Java Virtual Machine during class loading. They are not instantiated dynamically at runtime.
- **Scope:** Application-scoped. The enum constants exist for the entire lifetime of the application once the InteractionState class has been initialized by its class loader.
- **Destruction:** Instances are reclaimed by the JVM only when the application shuts down and its class loader is garbage collected.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant is a singleton whose internal state (the integer value) is final and set at compile time. It cannot be modified.
- **Thread Safety:** **Inherently thread-safe**. As immutable singletons, InteractionState constants can be safely passed between and read from any number of threads without requiring locks or other synchronization primitives.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | InteractionState | O(1) | Deserializes an integer from a network stream into an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary use case is within network packet handlers or game logic systems to interpret the result of a server-validated action.

```java
// Example of a client-side handler processing a server response
void handleInteractionResponse(int responseValue) {
    InteractionState state = InteractionState.fromValue(responseValue);

    switch (state) {
        case Finished:
            // Close the UI, the interaction was successful and complete
            uiManager.closeCurrentWindow();
            break;
        case ItemChanged:
            // The interaction succeeded, but the player's inventory changed
            inventory.requestSync();
            break;
        case Failed:
            // The server rejected the interaction, play a failure sound
            audioSystem.playSound(Sounds.INTERACTION_FAIL);
            break;
        // ... other cases
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Do not pass raw integer values (0, 1, 3, etc.) into game logic. This creates brittle code that is hard to read and maintain. Always convert to an InteractionState constant at the protocol boundary.
- **Bypassing Validation:** Do not implement custom deserialization logic that avoids calling fromValue. The bounds check within this method is a critical security and stability feature to protect against malformed packets.

## Data Pipeline
InteractionState is a data-transfer object that represents the terminal state of a client-server communication sequence.

> Flow:
> Server-side Game Logic -> **InteractionState** -> Protocol Serializer -> Network Packet -> Client Network Layer -> Protocol Deserializer -> **InteractionState** -> Client-side Game Logic / UI Update

