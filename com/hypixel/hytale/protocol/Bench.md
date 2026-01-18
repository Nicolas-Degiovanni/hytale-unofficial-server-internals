---
description: Architectural reference for Bench
---

# Bench

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class Bench {
```

## Architecture & Concepts
The Bench class is a Data Transfer Object (DTO) that operates within the Hytale network protocol layer. It is not a service or a manager; it is a pure data structure that represents the serialized state of a "bench" entity, likely a crafting station, for transmission between the client and server.

Its primary architectural function is to define a strict, low-level binary contract. The class encapsulates the logic for serializing its state into a Netty ByteBuf and deserializing from a ByteBuf back into a Java object. This design is optimized for network performance, employing techniques such as bitfields for nullability checks and variable-length integers (VarInt) to minimize payload size.

Bench objects are fundamental building blocks of higher-level network packets. They are not intended to hold business logic, but rather to serve as inert data containers that are created, transmitted, and consumed in a single, short-lived operation.

## Lifecycle & Ownership
- **Creation:** An instance of Bench is created under two primary circumstances:
    1.  **Deserialization:** The static `deserialize` method is invoked by a network channel handler (e.g., a Netty decoder) when a corresponding data structure is received from the network. This is the most common creation path on the receiving end.
    2.  **Direct Instantiation:** Game logic on the sending end (client or server) instantiates a new Bench object via its constructor to populate it with data before serialization.

- **Scope:** The lifecycle of a Bench object is extremely brief and transient. It is designed to exist only for the duration of its processing within a single network event or game tick.

- **Destruction:** The object becomes eligible for garbage collection as soon as the network handler or game logic method that created or received it completes its execution. No long-term references should ever be held to a Bench instance.

## Internal State & Concurrency
- **State:** The state of a Bench object is entirely defined by its public `benchTierLevels` array. The class is fully mutable, and its internal data can be directly accessed and modified after creation. It contains no caches, derived data, or internal logic beyond what is required for serialization.

- **Thread Safety:** **This class is not thread-safe.** It contains no synchronization primitives and its fields are publicly accessible. Instances are designed to be created, populated, serialized, and processed exclusively within a single thread, such as a Netty I/O worker thread or the main game loop thread.

    **WARNING:** Sharing a Bench instance across multiple threads without explicit, external locking will result in severe data corruption and unpredictable behavior. Do not pass a Bench object from a network thread to a worker thread if either thread can modify it.

## API Surface
The public API is focused exclusively on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static Bench | O(N) | Constructs a Bench object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. |
| computeBytesConsumed(ByteBuf, int) | static int | O(N) | Measures the size of a serialized Bench in a buffer without performing a full deserialization. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural integrity check on the buffered data without allocating a Bench object. |
| clone() | Bench | O(N) | Creates a deep copy of the Bench object and its contained data. |

*N = number of elements in benchTierLevels*

## Integration Patterns

### Standard Usage
The canonical use case involves creating a Bench object to send data or deserializing one to update game state.

```java
// SCENARIO: Sending bench data to the network
BenchTierLevel[] currentLevels = getLevelsFromGameState();
Bench benchData = new Bench(currentLevels);

// The buffer is typically provided by the network layer
ByteBuf networkBuffer = ...;
benchData.serialize(networkBuffer);
// The buffer is now ready for transmission


// SCENARIO: Receiving and processing bench data
ByteBuf receivedBuffer = ...;
Bench updatedBenchData = Bench.deserialize(receivedBuffer, 0);

// Immediately translate DTO data into the canonical game state
// Do not hold a reference to updatedBenchData
updateGameWorld(updatedBenchData.benchTierLevels);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not maintain instances of Bench as member variables in services, managers, or game state objects. It is a transient DTO, not a stateful model object. Its data should be copied into your application's canonical data structures immediately upon receipt.
- **Cross-Thread Sharing:** Never deserialize a Bench object on a network thread and pass the mutable instance to a game logic thread for processing. This is a classic race condition. Instead, create an immutable copy or a different, thread-safe data structure to pass between threads.

## Data Pipeline
The Bench class is a critical link in the data flow between the game engine and the network transport layer.

> **Outgoing Flow:**
> Game State -> **new Bench(data)** -> **serialize()** -> Netty ByteBuf -> Network Stack

> **Incoming Flow:**
> Network Stack -> Netty ByteBuf -> **deserialize()** -> **Bench instance** -> Game State Update -> Discarded

