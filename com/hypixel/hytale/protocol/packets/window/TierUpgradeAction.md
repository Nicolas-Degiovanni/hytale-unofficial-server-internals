---
description: Architectural reference for TierUpgradeAction
---

# TierUpgradeAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Transient

## Definition
```java
// Signature
public class TierUpgradeAction extends WindowAction {
```

## Architecture & Concepts
The TierUpgradeAction class is a concrete implementation of a network packet within the Hytale protocol stack. It functions as a Data Transfer Object (DTO) representing a specific, parameter-less command initiated by a user within a game window, such as a crafting station.

This class is part of a larger hierarchy inheriting from WindowAction, which standardizes all user interactions that can occur within interactive UI windows. Architecturally, it represents a **Command Pattern**, where the object itself encapsulates a request.

Its primary role is to act as a stateless signal from a client to the server. The presence of this packet in the data stream is the message; it carries no additional payload. The static constants such as FIXED_BLOCK_SIZE, all defined as zero, strongly suggest this class is part of a larger, possibly code-generated, serialization framework that accommodates both simple and complex packet structures. This specific action is one of the simplest, containing no variable data.

## Lifecycle & Ownership
- **Creation:** An instance of TierUpgradeAction is created under two distinct circumstances:
    1. **Client-Side (Sending):** Instantiated directly via its constructor (`new TierUpgradeAction()`) by UI event handlers when a player performs the corresponding action (e.g., clicking an upgrade button). The new object is then passed to the network subsystem for serialization.
    2. **Server-Side (Receiving):** Instantiated by the protocol decoding layer, which calls the static `deserialize` factory method after identifying the packet type from the incoming network buffer.

- **Scope:** The object's lifetime is exceptionally short. It is a transient object that exists only for the duration of a single processing tick or network event loop iteration. Once handled, it is immediately eligible for garbage collection.

- **Destruction:** There is no manual destruction logic. The Java Garbage Collector reclaims the memory once the object is no longer referenced by the network pipeline or game logic handler.

## Internal State & Concurrency
- **State:** TierUpgradeAction is **stateless and immutable**. It contains no instance fields and its behavior is fixed. It serves only as a type-safe signal.

- **Thread Safety:** This class is inherently thread-safe. As an immutable object, a single instance can be safely read by multiple threads without synchronization. However, in practice, instances are confined to a single network or game thread and are not shared. The static factory and utility methods operate on a provided ByteBuf and produce new instances, posing no concurrency risk to the class itself.

## API Surface
The public contract is designed for integration with a network protocol framework like Netty.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | TierUpgradeAction | O(1) | **[Factory]** Constructs a new instance from a network buffer. This is the designated entry point on the receiving end. |
| serialize(ByteBuf) | int | O(1) | Serializes the packet's (empty) state into the provided buffer. Returns the number of bytes written. |
| computeSize() | int | O(1) | Computes the size of the packet payload in bytes. For this class, it always returns 0. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs a low-level structural validation on a raw buffer to check for data integrity before deserialization. |

## Integration Patterns

### Standard Usage
This packet is intended to be processed by a server-side handler that receives a deserialized WindowAction. The handler uses type checking to delegate to the correct game logic.

```java
// Server-side packet handler
public void handleWindowAction(WindowAction action) {
    if (action instanceof TierUpgradeAction) {
        // Cast is safe due to the type check
        TierUpgradeAction upgradeAction = (TierUpgradeAction) action;
        
        // Retrieve player context and process the upgrade logic
        Player player = getPlayerForAction(action);
        player.getOpenWindow().executeTierUpgrade();
    }
    // ... other action types
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation on Receiver:** On the receiving end (e.g., the server), never create an instance with `new TierUpgradeAction()`. This bypasses the protocol layer's deserialization and buffer management. Always rely on the protocol decoder to call the static `deserialize` method.

- **Adding State:** Do not extend this class or modify it to include state. The protocol contract assumes this packet is a stateless signal. Adding data would require corresponding changes to the serialization, deserialization, and size computation methods, leading to protocol desynchronization.

## Data Pipeline
The flow of this object through the system is linear and unidirectional, typically from client to server.

> Flow:
> Client UI Event -> `new TierUpgradeAction()` -> Network Encoder (`serialize`) -> TCP Stream -> Server Network Decoder (`deserialize`) -> Game Logic Handler

