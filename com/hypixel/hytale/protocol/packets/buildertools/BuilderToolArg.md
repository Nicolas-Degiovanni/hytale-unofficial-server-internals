---
description: Architectural reference for BuilderToolArg
---

# BuilderToolArg

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class BuilderToolArg {
```

## Architecture & Concepts
The BuilderToolArg class is a fundamental data transfer object (DTO) within the Hytale network protocol, specifically designed to represent a single, typed argument for an in-game builder tool. It is not a service or a managed component but rather a structured container for data in transit.

Architecturally, this class implements a **tagged union** (or variant type) pattern. The field *argType* serves as the "tag," indicating which of the other nullable fields (e.g., boolArg, floatArg, intArg) holds the actual argument data. This design allows a single, consistent structure to encapsulate many different data types without resorting to polymorphism, which is often inefficient for network serialization.

The binary layout is heavily optimized for network efficiency. It consists of three main parts:
1.  A **Null Bitfield**: A 2-byte bitmask at the start of the structure that indicates which of the nullable fields are present. This allows for extremely fast checks during deserialization without reading the entire payload.
2.  A **Fixed-Size Block**: A 33-byte block containing primitive types and small, fixed-size objects. This ensures predictable read offsets for the most common data.
3.  A **Variable-Size Block**: A section at the end of the payload for variable-length data like strings. The fixed-size block contains offsets pointing to the start of each data element within this variable block.

This composite structure minimizes packet size and deserialization overhead, which is critical for real-time game performance.

## Lifecycle & Ownership
- **Creation:** A BuilderToolArg instance is created under two primary circumstances:
    1.  By the network protocol layer when deserializing an incoming packet that contains builder tool data. The static *deserialize* method is the entry point for this process.
    2.  By game logic on the client or server when constructing a new builder tool command to be sent over the network.
- **Scope:** The object's lifetime is ephemeral. It typically exists only for the duration of a single network packet's processing cycle. It is created, read by the relevant game system, and then becomes eligible for garbage collection.
- **Destruction:** There is no explicit destruction method. Instances are managed by the Java garbage collector and are reclaimed once they are no longer referenced, usually after the parent network packet has been fully handled.

## Internal State & Concurrency
- **State:** The BuilderToolArg is a mutable container. Its public fields are intended to be directly populated during deserialization or modified by game logic before serialization. The state is considered valid only when the *argType* field correctly corresponds to the single non-null argument field (e.g., if *argType* is BuilderToolArgType.Bool, only *boolArg* should be non-null).

- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single thread, such as a Netty event loop thread or the main game logic thread. Concurrent modification from multiple threads will lead to data corruption and unpredictable behavior. All access must be externally synchronized if multi-threaded access is unavoidable.

## API Surface
The primary API is static and focused on serialization and validation of raw byte buffers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static BuilderToolArg | O(N) | Constructs a BuilderToolArg instance by reading from a ByteBuf at a given offset. N is the size of the variable data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized BuilderToolArg directly from a buffer without full deserialization. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs structural validation on the data in a ByteBuf to ensure it represents a valid object before attempting deserialization. |
| clone() | BuilderToolArg | O(1) | Creates a shallow copy of the object. Note that the nested argument objects are also cloned. |

## Integration Patterns

### Standard Usage
A BuilderToolArg is never used in isolation. It is always part of a larger data structure, typically a list of arguments within a parent packet. Developers should interact with it via the parent object.

```java
// Example: Processing an argument from a hypothetical parent packet
void processBuilderToolPacket(BuilderToolPacket packet) {
    for (BuilderToolArg arg : packet.getArguments()) {
        switch (arg.argType) {
            case Bool:
                if (arg.boolArg != null) {
                    handleBooleanArgument(arg.boolArg.getValue());
                }
                break;
            case Int:
                if (arg.intArg != null) {
                    handleIntegerArgument(arg.intArg.getValue());
                }
                break;
            // ... other cases
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Do not manually construct an instance where the *argType* enum does not match the populated field. This will cause serialization failures or logic errors on the receiving end.
    ```java
    // BAD: Type is Bool, but the intArg field is populated
    BuilderToolArg arg = new BuilderToolArg();
    arg.argType = BuilderToolArgType.Bool;
    arg.intArg = new BuilderToolIntArg(10, 0, 100); // This will lead to bugs
    ```
- **Concurrent Modification:** Do not share a BuilderToolArg instance across threads without explicit locking. The internal state is not protected against race conditions.
- **Ignoring Null Checks:** The nullable argument fields **must** be checked for null before access. Relying solely on the *argType* field is unsafe, as a malformed packet could potentially violate this contract.

## Data Pipeline

The BuilderToolArg acts as a serialization and deserialization model for a segment of a network packet.

> **Outbound Flow (Serialization):**
> Game Logic -> Creates and populates **BuilderToolArg** -> `serialize()` -> Netty ByteBuf -> Network Stack

> **Inbound Flow (Deserialization):**
> Network Stack -> Netty ByteBuf -> `deserialize()` -> **BuilderToolArg** instance -> Consumed by Game Logic

