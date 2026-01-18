---
description: Architectural reference for AccumulationMode
---

# AccumulationMode

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public enum AccumulationMode
```

## Architecture & Concepts
AccumulationMode is a type-safe enumeration that defines a closed set of strategies for aggregating numerical data. Its primary architectural role is to serve as a contract between different systems, particularly between the network protocol layer and the core game logic.

By mapping specific strategies (Set, Sum, Average) to fixed integer values, it provides a highly efficient and unambiguous mechanism for serialization and deserialization. When a server sends a packet describing how a client-side statistic should be updated, it can transmit a single byte representing the mode. The client then uses this enum to translate that raw value back into a well-defined, self-documenting programmatic constant.

The inclusion of a custom ProtocolException during deserialization (`fromValue`) firmly places this enum as a critical component of the network protocol's data integrity and validation process. It prevents corrupted or malicious data from propagating into the game state.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during the class-loading phase. This process is managed entirely by the JVM and occurs before any game code is executed.
- **Scope:** Application-scoped. The instances (Set, Sum, Average) are static singletons that persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded only when the application terminates and the JVM shuts down. They are not subject to standard garbage collection.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a final integer value that is assigned at creation and can never be changed. The static VALUES array is also final and populated only once.
- **Thread Safety:** **Fully Thread-Safe**. Due to its immutable nature and the JVM's guarantees regarding enum initialization, this class can be safely accessed and used by any number of threads concurrently without requiring any external synchronization or locks.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant, intended for serialization. |
| fromValue(int value) | AccumulationMode | O(1) | A static factory method that converts a raw integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is intended to be used when decoding network packets or interpreting serialized game data. The `fromValue` method is the designated entry point for deserialization, and a switch statement is the canonical way to execute logic based on the result.

```java
// How a developer should normally use this
int modeFromPacket = readIntFromBuffer();
try {
    AccumulationMode mode = AccumulationMode.fromValue(modeFromPacket);
    switch (mode) {
        case Set:
            // Logic for setting a value directly
            break;
        case Sum:
            // Logic for adding to an existing value
            break;
        case Average:
            // Logic for averaging a value
            break;
    }
} catch (ProtocolException e) {
    // Handle data corruption or a protocol mismatch
    log.error("Received invalid AccumulationMode: " + modeFromPacket);
}
```

### Anti-Patterns (Do NOT do this)
- **Magic Numbers:** Never use the raw integer values (0, 1, 2) directly in game logic for comparisons. This defeats the purpose of type safety and creates brittle code.
    ```java
    // BAD
    if (modeFromPacket == 1) { /* ... */ }

    // GOOD
    if (mode == AccumulationMode.Sum) { /* ... */ }
    ```
- **Unsafe Deserialization:** Do not attempt to deserialize by accessing the static VALUES array directly. The `fromValue` method provides essential bounds checking that prevents ArrayOutOfBoundsException and surfaces data errors as a clear ProtocolException.
    ```java
    // BAD - Unsafe and bypasses validation
    AccumulationMode mode = AccumulationMode.VALUES[modeFromPacket];
    ```

## Data Pipeline
AccumulationMode acts as a translation point for data flowing from a raw, untyped transport layer to the strongly-typed game logic engine.

> Flow:
> Network Packet (Integer) -> Protocol Deserializer -> **AccumulationMode.fromValue()** -> Game Logic (AccumulationMode Constant) -> State Update

