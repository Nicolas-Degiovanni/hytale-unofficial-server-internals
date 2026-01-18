---
description: Architectural reference for BenchType
---

# BenchType

**Package:** com.hypixel.hytale.protocol
**Type:** Static Enumeration

## Definition
```java
// Signature
public enum BenchType {
```

## Architecture & Concepts
The BenchType enumeration is a foundational data type within the Hytale network protocol. It serves as a type-safe, compile-time constant representation for different crafting station identifiers. Its primary architectural function is to decouple game logic from the raw integer values used for network serialization.

Instead of passing "magic numbers" (e.g., 0 for Crafting, 1 for Processing) throughout the codebase, developers interact with the expressive and self-documenting BenchType constants like BenchType.Crafting. This pattern is critical for protocol maintainability and significantly reduces the risk of errors related to data corruption or version mismatches.

The class provides two core transformation functions:
1.  **Serialization:** Converting a high-level BenchType object into its low-level integer representation for transmission (via getValue).
2.  **Deserialization:** Converting a raw integer received from the network into a validated, safe BenchType object (via fromValue).

## Lifecycle & Ownership
- **Creation:** All instances of BenchType are created and initialized by the Java Virtual Machine (JVM) during class loading. This process is guaranteed to happen once, before any code can access the enumeration.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the JVM process. They are effectively global, immutable singletons.
- **Destruction:** The instances are reclaimed by the JVM only when the application shuts down. There is no manual destruction or garbage collection of enum constants during runtime.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a final integer value that is assigned at creation and can never be changed. The static VALUES array is also populated once during class initialization and is not modified thereafter.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature and the JVM's guarantees for static initialization, BenchType can be accessed from any thread without requiring locks or other synchronization primitives. It is safe for use in highly concurrent network and game logic threads.

## API Surface
The public contract of BenchType is focused exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | **Serialization.** Returns the low-level integer identifier associated with the enum constant, suitable for writing to a network buffer. |
| fromValue(int value) | BenchType | O(1) | **Deserialization.** Translates a raw integer from the network into a valid BenchType. This is a critical validation boundary. |

**WARNING:** The fromValue method will throw a ProtocolException if the provided integer does not map to a known BenchType. This is a security and stability feature to reject malformed or incompatible packets. Callers **must** handle this exception.

## Integration Patterns

### Standard Usage
The primary use case is to deserialize an integer from a network packet and then use the resulting enum in a switch statement to execute type-specific game logic.

```java
// Read an integer representing the bench type from a network buffer
int benchTypeId = buffer.readVarInt();

try {
    BenchType type = BenchType.fromValue(benchTypeId);

    // Use the type-safe enum to drive game logic
    switch (type) {
        case Crafting:
            // Handle standard crafting logic
            break;
        case StructuralCrafting:
            // Handle blueprint or structural crafting
            break;
        default:
            // Handle other cases
            break;
    }
} catch (ProtocolException e) {
    // A critical error occurred. The client sent an invalid ID.
    // This connection should be logged and potentially terminated.
    log.error("Received invalid BenchType ID: " + benchTypeId, e);
    connection.disconnect("Invalid protocol data");
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal() for Serialization:** Never use the built-in `ordinal()` method to get an integer for serialization. The value from `ordinal()` is dependent on the declaration order of the constants. If a new type is inserted in the middle, all subsequent ordinals will shift, breaking protocol compatibility. Always use the explicit `getValue()` method.
- **Ignoring ProtocolException:** Failing to catch the ProtocolException from `fromValue` will crash the processing thread. An unhandled invalid value from a malicious client could be used as a denial-of-service vector.

## Data Pipeline
BenchType acts as a translation point between the raw network layer and the abstract game logic layer.

> **Deserialization Flow:**
> Network Packet -> Byte Buffer -> Raw Integer -> **BenchType.fromValue(int)** -> Game Logic (Switch Statement)

> **Serialization Flow:**
> Game Logic (BenchType instance) -> **instance.getValue()** -> Raw Integer -> Byte Buffer -> Network Packet

