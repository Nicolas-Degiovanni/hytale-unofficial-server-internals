---
description: Architectural reference for Interaction
---

# Interaction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Model / Factory

## Definition
```java
// Signature
public abstract class Interaction {
```

## Architecture & Concepts

The Interaction class is the abstract base for Hytale's polymorphic interaction system. It serves a dual purpose: it acts as a factory for deserializing network data into concrete action objects, and it defines the common contract for all possible game interactions, such as breaking blocks, using entities, or modifying inventories.

Architecturally, this class is a critical component of the network protocol layer. It decouples the raw byte-level network processing from the higher-level game logic. When the server or client sends an interaction command, it is serialized into a byte stream. The first part of this stream is a VarInt type identifier, which this class uses to determine which concrete subclass to instantiate.

The extensive use of `switch` statements in static methods like `deserialize` and `computeBytesConsumed` is a deliberate design choice for performance and direct control over the binary protocol format. This pattern, known as a type dispatcher, avoids reflection or more complex factory registration systems, ensuring a fast and predictable deserialization path. Each case in the switch corresponds to a specific, concrete game action defined in the engine.

## Lifecycle & Ownership

- **Creation:** Interaction objects are exclusively created by the static `deserialize` method. This typically occurs within a Netty channel handler or a similar network message processing component when an incoming `ByteBuf` is decoded. They are never instantiated directly by game logic developers.
- **Scope:** The lifecycle of an Interaction instance is extremely short. It is a transient Data Transfer Object (DTO) that exists only for the duration of processing a single network packet or game event. Once the interaction has been processed by the relevant game system, the object is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** There is no manual destruction. The Java Garbage Collector reclaims memory once all references to the instance are dropped, which happens immediately after the interaction logic is executed.

## Internal State & Concurrency

- **State:** The base Interaction class and its subclasses hold mutable state that defines the parameters of a specific game action. For example, a `BreakBlockInteraction` would contain the coordinates of the block to be broken. This state is populated during deserialization and is considered read-only by the time it reaches game logic systems.
- **Thread Safety:** **Not thread-safe.** Interaction objects are designed to be created, processed, and discarded within a single thread, typically the main game loop or a Netty event loop thread. Sharing an Interaction instance across threads will lead to race conditions and unpredictable behavior. All processing must be synchronized or confined to a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Interaction | O(1) | Factory method. Reads a type ID from the buffer and delegates to the appropriate subclass deserializer. Throws ProtocolException for unknown types. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the total number of bytes a serialized Interaction occupies in the buffer without full deserialization. |
| getTypeId() | int | O(1) | Returns the unique integer identifier for the concrete subclass. Used for serialization. |
| serialize(buf) | abstract int | O(N) | Serializes the concrete interaction's data fields into the provided buffer. N depends on the subclass complexity. |
| serializeWithTypeId(buf) | int | O(N) | Writes the type ID followed by the serialized data of the instance into the buffer. This is the primary serialization entry point. |
| computeSize() | abstract int | O(N) | Calculates the byte size of the concrete interaction's data fields. |
| computeSizeWithTypeId() | int | O(N) | Calculates the total byte size of the serialized object, including its type ID. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a structural validation of the data in the buffer without full deserialization. Delegates to the appropriate subclass validator. |

## Integration Patterns

### Standard Usage

The `deserialize` method is the sole entry point for creating Interaction objects from network data. The resulting object is then passed to a handler or event bus for processing.

```java
// In a network message handler
ByteBuf incomingPacket = ...;

// The factory method determines the correct concrete type
Interaction action = Interaction.deserialize(incomingPacket, 0);

// Dispatch the action to the game logic engine
gameEngine.processInteraction(action);
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Never use `new SomeInteraction()`. The system relies on the static factory methods for correctness. Direct instantiation bypasses the protocol layer and will not produce valid objects for serialization.
- **State Modification After Deserialization:** Do not modify the fields of an Interaction object after it has been deserialized. Treat it as an immutable command object. Modifying its state can lead to desynchronization between the client and server.
- **Cross-Thread Access:** Never pass an Interaction object to another thread for processing without proper synchronization. These objects are not thread-safe and are intended for single-threaded execution contexts.

## Data Pipeline

The Interaction class is the central translation point in the data pipeline for game actions, converting low-level network bytes into high-level, executable game commands.

> Flow:
> Network ByteBuf -> **Interaction.deserialize** -> Concrete Interaction Object (e.g., BreakBlockInteraction) -> Game Event Bus -> Interaction Handling System -> World State Mutation

