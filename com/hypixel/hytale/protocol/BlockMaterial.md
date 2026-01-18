---
description: Architectural reference for BlockMaterial
---

# BlockMaterial

**Package:** com.hypixel.hytale.protocol
**Type:** Value Type (Enum)

## Definition
```java
// Signature
public enum BlockMaterial {
```

## Architecture & Concepts

The BlockMaterial enum is a foundational type-safe constant within the Hytale protocol layer. It represents the fundamental physical property of a block—whether it is empty space or a solid object. Its primary architectural role is to provide a robust, efficient, and unambiguous representation for this state during network serialization and deserialization.

By mapping the abstract concepts of *Empty* and *Solid* to fixed integer values (0 and 1), the system optimizes for network bandwidth. Sending a single byte is significantly more efficient than transmitting a descriptive string.

The static factory method, fromValue, acts as a critical validation boundary. When data is received from a network stream, this method is responsible for converting the raw integer back into a valid BlockMaterial instance. Its inclusion of strict bounds checking, which throws a ProtocolException for invalid values, is a key defensive programming pattern that prevents corrupted or malicious data from propagating into the core game state.

## Lifecycle & Ownership

As a Java enum, BlockMaterial has a lifecycle strictly managed by the Java Virtual Machine (JVM), not by application-level code.

-   **Creation:** The enum constants, Empty and Solid, are instantiated automatically by the JVM's class loader the first time the BlockMaterial class is referenced. They are effectively static singletons.
-   **Scope:** The instances persist for the entire lifetime of the application. They are globally accessible and shared across all threads.
-   **Destruction:** The enum instances are garbage collected only when the application's class loader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency

-   **State:** The BlockMaterial enum is **immutable**. The internal integer value for each constant is final and assigned only once during its JVM-managed instantiation. The VALUES array is also static and final, providing a read-only cache of the enum's constants.
-   **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that its state cannot be modified after construction, eliminating the need for any external synchronization or locking mechanisms. It can be safely read and passed between any number of concurrent threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation for network serialization. |
| fromValue(int value) | BlockMaterial | O(1) | **Deserialization Entry Point.** Converts an integer from a network packet into a BlockMaterial instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage

The primary use case is deserializing a block's material property from a network buffer or data stream. The fromValue method is the designated entry point for this operation.

```java
// Reading a material type from a network packet
int materialId = packet.readByte();
BlockMaterial material = BlockMaterial.fromValue(materialId);

// Use the type-safe enum in game logic
if (material == BlockMaterial.Solid) {
    // Handle collision
}
```

### Anti-Patterns (Do NOT do this)

-   **Using Magic Numbers:** Do not use raw integers (0, 1) in game logic. The entire purpose of this enum is to provide type safety and readability. Rely on the enum constants directly.
-   **Ignoring Deserialization Errors:** Swallowing the ProtocolException thrown by fromValue can lead to a desynchronized or corrupted game state. Invalid data from the network must be handled, typically by disconnecting the offending client.

## Data Pipeline

BlockMaterial serves as a translation point between the high-level game state and the low-level network protocol.

> **Serialization Flow:**
> Game World Block State -> `block.getMaterial().getValue()` -> **Integer (0 or 1)** -> Network Packet Buffer

> **Deserialization Flow:**
> Network Packet Buffer -> **Integer (e.g., 1)** -> `BlockMaterial.fromValue(1)` -> **BlockMaterial.Solid** -> Game World Block Update

