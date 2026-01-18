---
description: Architectural reference for ConditionInteraction
---

# ConditionInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Object

## Definition
```java
// Signature
public class ConditionInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The ConditionInteraction class is a specialized data structure within the Hytale network protocol, representing a conditional node in the game's interaction state machine. It extends SimpleInteraction, inheriting a baseline structure, but adds a layer of conditional logic based on the player's current state.

Its primary role is to enable or disable subsequent interactions by evaluating a set of boolean player states (e.g., *is the player flying?*, *is the player crouching?*) and the current GameMode. Based on the outcome of this evaluation, the interaction system is directed to transition to one of two subsequent interactions, identified by the **next** (success) or **failed** (failure) integer fields.

Architecturally, this class is a pure data container designed for extremely efficient network serialization and deserialization. The binary layout is highly optimized:

1.  **Null Bit Field:** A 2-byte bitmask at the start of the payload indicates which of the many nullable fields are present in the data stream, eliminating the need for explicit null terminators or presence flags for each field.
2.  **Fixed-Size Block:** A 26-byte block contains predictable, fixed-width fields like multipliers, state booleans, and state machine transition IDs. This allows for constant-time access during deserialization.
3.  **Variable-Size Block:** Pointers (offsets) within the fixed block reference the start of complex, variable-length data structures (like maps and arrays) which are appended after the fixed block. This hybrid model provides both the performance of fixed layouts and the flexibility of dynamic data.

This class is fundamental to creating dynamic and context-aware gameplay sequences, such as quests, tutorials, or environmental interactions, without requiring bespoke server-side logic for every condition.

## Lifecycle & Ownership

-   **Creation:** ConditionInteraction instances are almost exclusively created by the network protocol layer. The static **deserialize** method is invoked by a packet handler when a corresponding data structure is received from the server. Manual instantiation via its constructor is reserved for server-side logic or development tools that build interaction graphs.
-   **Scope:** The object's lifetime is exceptionally short and tied to a single scope of processing. It is deserialized, immediately evaluated by the client's interaction system, and then becomes eligible for garbage collection. It is not intended to be cached or referenced over a long period.
-   **Destruction:** The object is managed by the Java Garbage Collector. Once the interaction system has processed it and all references are dropped, it is destroyed. There is no manual memory management or destruction required.

## Internal State & Concurrency

-   **State:** The class is a mutable data container. All of its fields are public and can be directly modified after creation. This design prioritizes performance and ease of use within the single-threaded context of the protocol and game loop.
-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. It is designed to be deserialized on a network thread (e.g., Netty worker) and immediately passed to a single consumer thread (e.g., the main game thread) for processing. Concurrent reads or writes from multiple threads will lead to undefined behavior and data corruption. All access must be externally synchronized if multi-threaded use is unavoidable.

## API Surface

The primary API surface consists of static methods for serialization, deserialization, and validation, reflecting its role as a protocol data structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ConditionInteraction | O(N) | **Primary Factory.** Constructs an object by reading from a Netty ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. Returns the number of bytes written. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the binary data in a buffer without full deserialization. Crucial for preventing crashes from malformed packets. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a serialized object directly from a buffer. Used by protocol handlers to skip over data streams. |
| clone() | ConditionInteraction | O(N) | Creates a deep copy of the object, including its nested collections and data structures. |

## Integration Patterns

### Standard Usage

The canonical use case involves a protocol handler decoding a network buffer and passing the resulting object to a state machine or interaction service for evaluation.

```java
// Executed by a network packet handler
ByteBuf packetData = ...;

// Validate before processing to ensure protocol integrity
ValidationResult result = ConditionInteraction.validateStructure(packetData, 0);
if (!result.isValid()) {
    throw new ProtocolException("Invalid ConditionInteraction received: " + result.error());
}

// Deserialize into a transient object
ConditionInteraction interaction = ConditionInteraction.deserialize(packetData, 0);

// Pass the object to the game's interaction system for processing
// This typically happens on the main game thread
InteractionSystem.process(interaction);
```

### Anti-Patterns (Do NOT do this)

-   **Long-Term Storage:** Do not hold references to ConditionInteraction objects in caches or long-lived components. They represent a point-in-time state from the server and should be processed immediately.
-   **Cross-Thread Modification:** Do not deserialize an object on one thread and modify its public fields on another without explicit locking. This will cause severe race conditions.
-   **Ignoring Validation:** Never deserialize data from an untrusted source (i.e., the network) without first calling **validateStructure**. Bypassing this step exposes the client to buffer overflows and other parsing vulnerabilities.

## Data Pipeline

The flow of data through this component is linear and unidirectional, from the network socket to the game logic.

> Flow:
> Network ByteBuf -> Protocol Decoder -> **ConditionInteraction.deserialize()** -> ConditionInteraction Instance -> Interaction State Machine -> Game State Transition (Success/Failure)

