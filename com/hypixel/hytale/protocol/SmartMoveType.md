---
description: Architectural reference for SmartMoveType
---

# SmartMoveType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum SmartMoveType {
```

## Architecture & Concepts
The SmartMoveType enum provides a type-safe, fixed set of constants representing high-level inventory management actions initiated by the player. These actions, often triggered by shortcuts like shift-clicking, require server-side logic to determine the final destination of an item.

This enum serves as a critical component in the client-server communication protocol. It translates a complex user intention into a compact, serializable integer. This design pattern is fundamental for network performance, as it avoids transmitting verbose string identifiers, minimizing packet size.

The class includes a reverse lookup factory method, fromValue, which acts as a deserialization and validation gate. Any integer received from the network that does not correspond to a valid SmartMoveType will be rejected by throwing a ProtocolException, thus preventing the propagation of corrupt or malicious data into the server's game logic.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the Java Virtual Machine during class loading. This occurs at application startup, before any game-specific code is executed.
- **Scope:** Application-wide. As static singletons, these constants persist for the entire lifetime of the application.
- **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual memory management.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final integer value that is assigned at creation and can never be changed. The static VALUES array is also effectively immutable after its initial population.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature guarantees that it can be safely accessed and read from any thread without the need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer value for serialization. |
| fromValue(int value) | SmartMoveType | O(1) | Deserializes an integer into an enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of network packets related to inventory actions.

```java
// Serialization: Writing the enum to a network buffer
int moveId = SmartMoveType.EquipOrMergeStack.getValue();
packetBuffer.writeVarInt(moveId);

// Deserialization: Reading the enum from a network buffer
int receivedId = packetBuffer.readVarInt();
try {
    SmartMoveType moveType = SmartMoveType.fromValue(receivedId);
    // Process the valid inventory move
    handleInventoryAction(moveType);
} catch (ProtocolException e) {
    // Disconnect client for sending invalid data
    log.error("Invalid SmartMoveType received: " + receivedId);
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method for serialization. The integer value returned by `ordinal()` is dependent on the declaration order of the constants. If a new constant is added or the order is changed, all previously serialized data will become invalid. Always use the explicit `getValue()` method.
- **Ignoring Exceptions:** The ProtocolException thrown by `fromValue` is a critical security and stability feature. It must be caught to handle invalid data from a client. Failure to do so can crash the packet processing thread.

## Data Pipeline
SmartMoveType acts as a data contract within the inventory network pipeline, ensuring that client intent is understood by the server.

> Flow:
> Client Input (Shift-Click) -> UI Logic maps to **SmartMoveType** -> Packet Serializer calls `getValue()` -> Network Packet -> Server Packet Deserializer calls `fromValue()` -> Server Inventory System receives **SmartMoveType** instance

