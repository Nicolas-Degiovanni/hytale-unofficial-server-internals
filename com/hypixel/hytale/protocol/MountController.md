---
description: Architectural reference for MountController
---

# MountController

**Package:** com.hypixel.hytale.protocol
**Type:** Utility (Enumeration)

## Definition
```java
// Signature
public enum MountController {
```

## Architecture & Concepts
MountController is a type-safe enumeration that defines the contract for different mount control schemes within the Hytale network protocol. Its primary architectural role is to translate low-level integer identifiers, received from network packets, into high-level, self-documenting constants for use within the game engine.

This class acts as a strict validation and deserialization boundary. By encapsulating the integer-to-enum mapping, it prevents the propagation of invalid "magic numbers" into the core game logic. The static factory method, fromValue, serves as the designated gateway for all incoming network data related to mount types, immediately rejecting any values that do not conform to the protocol specification by throwing a ProtocolException.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. This process is guaranteed to happen only once for the entire application lifecycle.
- **Scope:** As JVM-managed singletons, all MountController constants (Minecart, BlockMount) persist for the lifetime of the application. They are loaded into memory and are never eligible for garbage collection until the defining ClassLoader is unloaded.
- **Destruction:** Not applicable. Enum instances are not manually destroyed.

## Internal State & Concurrency
- **State:** Immutable. The internal integer state of each enum constant is assigned via the private constructor during class loading and is declared final. It cannot be modified at runtime.
- **Thread Safety:** Inherently thread-safe. As immutable singletons managed by the JVM, MountController constants can be safely accessed and read from any thread without synchronization. The static VALUES array is also safely published by the JVM's class loading mechanism.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for serialization over the network. |
| fromValue(int value) | MountController | O(1) | **Critical Deserialization Method.** Converts an integer from a network packet into a MountController instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is almost exclusively used during the network packet serialization and deserialization process.

**Deserialization (Reading from a packet):**
```java
// packetBuffer contains the raw integer ID for the mount type
int mountTypeId = packetBuffer.readVarInt();

// Convert the ID to a type-safe enum, handling potential protocol errors
try {
    MountController controller = MountController.fromValue(mountTypeId);
    // ... use the controller in game logic, e.g., entity.setMountController(controller);
} catch (ProtocolException e) {
    // Handle a corrupt or malicious packet. Typically results in disconnecting the client.
    log.error("Received invalid MountController ID: " + mountTypeId, e);
    connection.disconnect("Invalid mount data");
}
```

**Serialization (Writing to a packet):**
```java
// Get the controller from a game object
MountController controller = entity.getMountController();

// Write its integer representation to the network buffer
packetBuffer.writeVarInt(controller.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Exceptions:** Failure to catch the ProtocolException from fromValue is a severe defect. It allows invalid data to bypass the protocol validation layer, which can lead to server crashes or unpredictable game state corruption.
- **Manual Comparison:** Do not compare mount types using their raw integer values. This defeats the purpose of a type-safe enum and reintroduces magic numbers.
    - **BAD:** `if (entity.getMountController().getValue() == 0)`
    - **GOOD:** `if (entity.getMountController() == MountController.Minecart)`

## Data Pipeline
The MountController acts as a deserialization and validation stage in the inbound network data pipeline.

> Flow:
> Network Packet (containing integer ID) -> Protocol Packet Decoder -> **MountController.fromValue(id)** -> Game Entity System -> Player State Update

