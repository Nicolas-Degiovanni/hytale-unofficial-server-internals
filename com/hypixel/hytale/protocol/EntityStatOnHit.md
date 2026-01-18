---
description: Architectural reference for EntityStatOnHit
---

# EntityStatOnHit

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class EntityStatOnHit {
```

## Architecture & Concepts

The EntityStatOnHit class is a low-level Data Transfer Object (DTO) used within the Hytale network protocol layer. It is not a service or a manager, but rather a structured representation of a specific game mechanic: a statistical modification applied to an entity upon being hit. Its primary purpose is to facilitate the exchange of this game state information between the client and the server.

This class acts as a serialization contract. It defines a precise binary layout for an "on-hit" effect, including a base amount, a list of multipliers for successive targets (e.g., for cleaving or piercing attacks), and a fallback multiplier. The design prioritizes network efficiency through a compact binary format that utilizes a bitfield for nullable members and variable-length integers (VarInts) for array counts.

It is a fundamental component of the combat data pipeline, directly mapping game logic concepts to a transmissible byte-level format.

## Lifecycle & Ownership

-   **Creation:** Instances of EntityStatOnHit are ephemeral and created in two primary scenarios:
    1.  **Serialization (Outgoing):** The game logic, typically a server-side combat system, instantiates and populates an EntityStatOnHit object when an on-hit event needs to be communicated to a client.
    2.  **Deserialization (Incoming):** The static factory method *deserialize* is invoked by a network channel handler (e.g., a Netty pipeline component) when a corresponding data block is read from an incoming ByteBuf.

-   **Scope:** The object's lifetime is extremely short. It exists only for the duration of a single processing task—either to be written to a network buffer or to be read by a game system immediately after being deserialized. It is not intended to be stored or referenced long-term.

-   **Destruction:** The object is managed by the Java Garbage Collector. Once all references are dropped, which typically happens after the network operation or game logic update is complete, it becomes eligible for collection. There are no manual resource management requirements.

## Internal State & Concurrency

-   **State:** The internal state is fully **mutable**. All data fields are public, allowing for direct modification after instantiation. The class is a simple data container and performs no internal caching or state transformation beyond what is required for serialization.

-   **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms. It is designed to be created, populated, and processed within a single-threaded context, such as a Netty I/O thread or the main game loop.

    **Warning:** Sharing an instance of EntityStatOnHit across multiple threads without external locking will result in race conditions, data corruption, and non-deterministic behavior. Do not write to an instance from one thread while another thread is serializing or reading from it.

## API Surface

The public API is centered on serialization, deserialization, and size calculation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static EntityStatOnHit | O(N) | Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data or buffer underflow. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the defined binary protocol. |
| computeSize() | int | O(1) | Calculates the number of bytes this object will occupy when serialized. Does not account for the array itself. |
| computeBytesConsumed(buf, offset) | static int | O(1) | Calculates the size of a serialized object directly from a buffer without full deserialization. |
| validateStructure(buffer, offset) | static ValidationResult | O(1) | Performs a lightweight check on a buffer to verify if it likely contains a valid object structure. |
| clone() | EntityStatOnHit | O(N) | Creates a deep copy of the object. N is the length of the multipliers array. |

*Complexity O(N) refers to the number of elements in the multipliersPerEntitiesHit array.*

## Integration Patterns

### Standard Usage

The class is almost exclusively used by the protocol layer. A network handler receives a buffer, identifies the data type, and uses the static deserialize method to rehydrate the object for consumption by the game logic.

```java
// Executed within a Netty channel handler or similar context
public void processCombatPacket(ByteBuf packetData) {
    // Assume offset points to the start of the EntityStatOnHit data
    int offset = ...;

    // Validate before attempting to deserialize
    ValidationResult result = EntityStatOnHit.validateStructure(packetData, offset);
    if (!result.isOk()) {
        throw new MalformedPacketException(result.getErrorMessage());
    }

    // Deserialize the object from the buffer
    EntityStatOnHit onHitEffect = EntityStatOnHit.deserialize(packetData, offset);

    // Pass the rehydrated DTO to the game logic for processing
    gameCombatSystem.applyOnHitEffect(this.entity, onHitEffect);
}
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not modify and re-serialize the same EntityStatOnHit instance for multiple distinct game events. These objects are lightweight; instantiate a new one for each message to prevent state leakage and subtle bugs.

-   **Manual Deserialization:** Do not attempt to read fields from a ByteBuf manually. The static *deserialize* method is the canonical entry point and correctly handles the null-bitfield, VarInt decoding, and bounds checking. Bypassing it will lead to data corruption.

-   **Long-Term Storage:** Do not hold references to EntityStatOnHit objects in long-lived components or caches. They represent a point-in-time event and should be processed and discarded immediately.

## Data Pipeline

EntityStatOnHit serves as a data payload within the network pipeline. It translates a high-level game event into a binary representation and back again.

> **Flow (Server to Client):**
> Game Event (e.g., Player hits multiple mobs) -> Server Combat System creates **EntityStatOnHit** -> Protocol Serializer writes it to a ByteBuf -> TCP/IP Stack -> Client Network Handler reads ByteBuf -> Protocol Deserializer rehydrates **EntityStatOnHit** -> Client Game Logic applies visual/audio feedback.

