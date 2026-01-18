---
description: Architectural reference for BlockMountType
---

# BlockMountType

**Package:** com.hypixel.hytale.protocol
**Type:** Utility / Enum

## Definition
```java
// Signature
public enum BlockMountType {
```

## Architecture & Concepts
The BlockMountType enum provides a type-safe, compile-time constant representation for ways an entity can interact with or "mount" a block. Its primary architectural role is to serve as a translation layer between raw integer identifiers used in the network protocol and self-documenting, readable constants used within the game logic.

This class is a foundational element of the protocol deserialization pipeline. By encapsulating the mapping from an integer to a specific mount type, it prevents the proliferation of "magic numbers" throughout the codebase. The static factory method, fromValue, acts as a validation gatekeeper. It ensures that any integer received from the network corresponds to a known, valid mount type. If an invalid value is encountered, it throws a specific ProtocolException, allowing the network layer to gracefully handle data corruption or protocol mismatches.

## Lifecycle & Ownership
-   **Creation:** All enum constants (Seat, Bed) and the internal VALUES array are instantiated once by the JVM during static class initialization. This process is automatic and occurs before any code can access the enum.
-   **Scope:** Application-wide. The enum constants are static final members of the class and persist for the entire lifetime of the application.
-   **Destruction:** The constants are garbage collected only when the class loader is unloaded, which typically happens only on JVM shutdown. No manual cleanup is ever required.

## Internal State & Concurrency
-   **State:** Immutable. The state of each enum constant, including its name and associated integer value, is fixed at compile time. The internal VALUES array is populated once during class loading and is not modified thereafter.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable nature guarantees that it can be safely read and passed between any number of threads without external synchronization or locking mechanisms. The static fromValue method is a pure function, and its operations are atomic, making it safe for concurrent invocation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant, used for serialization. |
| fromValue(int value) | BlockMountType | O(1) | **Static Factory.** Deserializes an integer into a BlockMountType instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet deserialization to convert a network integer into a usable game object.

```java
// In a packet handler or deserializer
int mountTypeValue = packet.readVarInt();
try {
    BlockMountType mountType = BlockMountType.fromValue(mountTypeValue);

    // Pass the type-safe enum to the game logic
    player.mountBlock(targetBlock, mountType);
} catch (ProtocolException e) {
    // Handle corrupt or invalid packet data
    connection.disconnect("Invalid mount type received.");
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Value Comparison:** Never use the raw integer value for logical comparisons. This defeats the purpose of a type-safe enum and reintroduces magic numbers.

    ```java
    // BAD: Prone to error if values change
    if (mountType.getValue() == 0) {
        // Logic for seating
    }

    // GOOD: Type-safe and self-documenting
    if (mountType == BlockMountType.Seat) {
        // Logic for seating
    }
    ```
-   **Ignoring Exceptions:** The ProtocolException thrown by fromValue is a critical signal of invalid data. It must not be caught and ignored, as this can lead to undefined behavior or security vulnerabilities. The connection handler should treat this as a terminal error for the packet being processed.

## Data Pipeline
BlockMountType acts as a deserialization and validation step within the data pipeline. It transforms raw network data into a validated, structured object for consumption by higher-level game systems.

> Flow:
> Network Byte Stream -> Protocol Deserializer -> `BlockMountType.fromValue(intValue)` -> **BlockMountType Instance** -> Game Logic / Player State Machine

