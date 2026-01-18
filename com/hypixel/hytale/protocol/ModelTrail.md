---
description: Architectural reference for ModelTrail
---

# ModelTrail

**Package:** com.hypixel.hytale.protocol
**Type:** Protocol Model / DTO

## Definition
```java
// Signature
public class ModelTrail {
```

## Architecture & Concepts

The ModelTrail class is a fundamental Data Transfer Object (DTO) within the Hytale network protocol. It is not a service or manager, but rather a pure data structure that represents the complete configuration for a visual trail effect attached to an in-game entity. Its primary architectural role is to serve as the canonical, language-neutral representation of a trail for serialization and deserialization across the network.

The binary format is highly optimized for performance, eschewing standard formats like JSON or XML in favor of a custom layout. This layout is a critical design choice and follows a common high-performance pattern:

1.  **Fixed-Size Block:** The first 35 bytes of the serialized object contain fixed-width data. This includes a crucial **null bitfield** (a single byte acting as a set of flags to indicate which nullable fields are present), enums, booleans, and embedded fixed-size structures like Vector3f and Direction.
2.  **Variable-Size Block:** Following the fixed block is a data region for variable-length fields, primarily strings like trailId and targetNodeName.
3.  **Offset Pointers:** The fixed-size block contains integer offsets that point to the start of each corresponding field within the variable-size block. This design allows for extremely fast reads of the fixed-size data, as a parser can jump directly to any field without needing to parse preceding variable-length data.

This structure ensures both efficient use of network bandwidth and low-latency parsing on the client and server.

## Lifecycle & Ownership

-   **Creation:** A ModelTrail instance is created under two primary circumstances:
    1.  **Deserialization:** The network layer instantiates it via the static `deserialize` factory method when processing an incoming network packet.
    2.  **Game Logic:** The server or client game logic creates a new instance via its constructor to define a trail that needs to be synchronized over the network or applied to a local entity.
-   **Scope:** Instances are ephemeral and short-lived. Their scope is typically confined to the processing of a single network packet or a single frame of game logic. They are passed by value (or reference, in Java) and are not intended to be stored as long-term state.
-   **Destruction:** The object is managed entirely by the Java Garbage Collector. There are no manual resource management methods. Once all references to an instance are dropped (e.g., after a network packet has been fully processed), it becomes eligible for garbage collection.

## Internal State & Concurrency

-   **State:** The class is fully **mutable**, with public fields for direct access. This is a deliberate performance-oriented design choice to avoid the overhead of getter and setter method calls in performance-critical code paths like the network or game loop.
-   **Thread Safety:** ModelTrail is **not thread-safe**. It is designed for use within a single, well-defined thread, such as a Netty I/O thread or the main game logic thread. Modifying an instance from one thread while another thread is serializing or reading it will result in data corruption and undefined behavior. All synchronization must be handled externally by the calling system.

## API Surface

The public API is centered around serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ModelTrail | O(N) | Constructs a ModelTrail instance by reading from a ByteBuf. N is the total length of string data. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf according to the defined binary protocol. N is the total length of string data. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a low-cost check on a buffer to verify that the data appears to be a valid ModelTrail without full deserialization. Critical for security and stability. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the total size of a serialized ModelTrail within a buffer by reading its header and offset pointers. Does not perform a full parse. |
| computeSize() | int | O(N) | Calculates the byte size the current object instance will occupy when serialized. N is the total length of string data. |
| clone() | ModelTrail | O(1) | Creates a copy of the object. Nested objects like Vector3f are also cloned. |

## Integration Patterns

### Standard Usage

The primary use case is serializing data for network transmission or deserializing received data into a usable in-memory object.

```java
// SCENARIO 1: Deserializing an incoming trail from a network buffer
ValidationResult result = ModelTrail.validateStructure(incomingBuffer, offset);
if (result.isOk()) {
    ModelTrail trail = ModelTrail.deserialize(incomingBuffer, offset);
    // Use the trail data in the entity or rendering system
    applyTrailToEntity(player, trail);
}

// SCENARIO 2: Creating and serializing a new trail to send
ModelTrail newTrail = new ModelTrail(
    "hytale:trail.fire",
    EntityPart.RightHand,
    "WeaponNode",
    null,
    null,
    false
);
newTrail.serialize(outgoingBuffer);
```

### Anti-Patterns (Do NOT do this)

-   **Instance Re-use:** Do not modify and re-serialize the same ModelTrail instance for different logical packets. These objects are cheap to create and should be treated as immutable once created or after deserialization. Re-using instances can lead to subtle bugs where old state is unintentionally carried over.
-   **Concurrent Access:** Never share a ModelTrail instance between threads without explicit and robust external locking. Do not, for example, have a game logic thread modifying a trail while a network thread is simultaneously calling `serialize` on it.
-   **Skipping Validation:** On untrusted input (e.g., packets from a client), never call `deserialize` without first calling `validateStructure`. Bypassing validation exposes the server to malformed packets that could trigger exceptions and potentially cause denial-of-service vulnerabilities.

## Data Pipeline

ModelTrail acts as a data container that flows between the network protocol layer and the core game simulation/rendering systems.

> **Ingress (Server/Client Receiving Data):**
> Raw ByteBuf -> `ModelTrail.validateStructure` -> `ModelTrail.deserialize` -> **ModelTrail Instance** -> Entity Component System -> Rendering Engine

> **Egress (Server/Client Sending Data):**
> Game Event (e.g., Player equips item) -> Game Logic creates **ModelTrail Instance** -> `ModelTrail.serialize` -> Raw ByteBuf -> Network Layer (Netty)

