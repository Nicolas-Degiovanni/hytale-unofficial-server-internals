---
description: Architectural reference for BlockPlacementRotationMode
---

# BlockPlacementRotationMode

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum BlockPlacementRotationMode {
```
This enumeration implicitly extends `java.lang.Enum`.

## Architecture & Concepts
BlockPlacementRotationMode is a fundamental component of the Hytale network protocol layer. Its primary architectural role is to provide a type-safe, robust, and human-readable contract for defining how a block's orientation should be calculated upon placement in the world.

It serves as a translation layer between high-level game logic (e.g., "place this stair facing the player") and the low-level integer representation required for efficient network transmission. By encapsulating these placement strategies into a fixed set of constants, the system avoids the use of "magic numbers", thereby reducing the risk of desynchronization errors between the client and server.

The existence of specialized modes like `StairFacingPlayer` indicates that the block placement system is not monolithic; it contains conditional logic that depends on the type of block being placed, and this enum is the primary discriminant for that logic.

### Lifecycle & Ownership
- **Creation:** All instances of this enum are created and initialized by the Java Virtual Machine (JVM) during class loading. They are compile-time constants, not runtime objects created by application code.
- **Scope:** Application-scoped. The enum constants exist for the entire lifetime of the application process.
- **Destruction:** The instances are reclaimed by the garbage collector only when the defining class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** BlockPlacementRotationMode is **deeply immutable**. Each enum constant holds a final `value` field that is set at creation and can never be changed. The static `VALUES` array, a common performance optimization for enums, is populated once during class initialization and is not modified thereafter.
- **Thread Safety:** This class is **inherently thread-safe**. Due to its immutability, instances can be safely shared and read across any number of threads without requiring locks or other synchronization primitives. This guarantee is critical for components within the multithreaded networking and game logic pipelines.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer value for network serialization. |
| fromValue(int value) | BlockPlacementRotationMode | O(1) | Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the serialization and deserialization of packets related to player actions, such as placing a block.

```java
// Example: Deserializing a block placement packet
try {
    int modeId = packetBuffer.readVarInt();
    BlockPlacementRotationMode rotationMode = BlockPlacementRotationMode.fromValue(modeId);

    // Pass the resolved mode to the world placement system
    world.placeBlock(position, blockType, rotationMode);
} catch (ProtocolException e) {
    // A malformed packet was received. This is a critical error.
    connection.disconnect("Invalid block placement rotation mode.");
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinal for Serialization:** Do **not** use the built-in `ordinal()` method for serialization. The integer `value` field is explicitly defined for a stable network contract. The `ordinal()` value is fragile and will change if the declaration order of the enum constants is modified, leading to catastrophic protocol mismatches.
- **Ignoring ProtocolException:** The `fromValue` method throws a checked exception for a reason. An invalid value indicates a corrupted packet or a protocol version mismatch between the client and server. Swallowing this exception will lead to undefined behavior and state corruption. The connection should be terminated.

## Data Pipeline
BlockPlacementRotationMode acts as a marshalling and unmarshalling component in the data flow between game logic and the network layer.

> **Outbound Flow (e.g., Client to Server):**
> Player Input -> Game Logic determines `BlockPlacementRotationMode.FacingPlayer` -> Packet Serializer calls `getValue()` -> `0` is written to Network Buffer -> TCP/IP Stack

> **Inbound Flow (e.g., Server from Client):**
> TCP/IP Stack -> Network Buffer with value `0` -> Packet Deserializer calls `fromValue(0)` -> **BlockPlacementRotationMode.FacingPlayer** -> World Simulation Logic

