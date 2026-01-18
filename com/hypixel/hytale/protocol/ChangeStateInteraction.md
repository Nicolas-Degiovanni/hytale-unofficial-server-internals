---
description: Architectural reference for ChangeStateInteraction
---

# ChangeStateInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ChangeStateInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The ChangeStateInteraction class is a specialized Data Transfer Object (DTO) operating within the network protocol layer. It represents a specific, serializable game event: an interaction that results in one or more state changes to a game object, typically a block or entity. It inherits from SimpleBlockInteraction, extending a base set of interaction mechanics with a flexible key-value map for state modification.

Its primary architectural role is to serve as an inert data container that is efficiently encoded into a byte stream for network transmission and decoded back into an object on the receiving end. The class itself contains no logic; it is a pure data structure.

The serialization format is a critical aspect of its design, optimized for performance and minimal network overhead. It employs a hybrid fixed-size and variable-size layout:
1.  **Fixed-Size Block (44 bytes):** Contains primitive fields and a one-byte bitmask (*nullBits*) indicating the presence of optional, variable-sized fields.
2.  **Offset Table:** Within the fixed block, a series of integer slots store the relative offsets to the start of each variable-sized field's data.
3.  **Variable-Size Block:** A contiguous data region following the fixed block where all complex data (maps, arrays, nested objects) is written.

This structure allows for extremely fast validation and partial reads, as a consumer can calculate the total size and locate specific fields without parsing the entire payload.

### Lifecycle & Ownership
- **Creation:**
    - **Server-Side (Serialization Path):** Instantiated directly via its constructor (`new ChangeStateInteraction(...)`) by game logic systems when an interaction needs to be communicated to a client. The object is populated with data and then passed to the network layer for serialization.
    - **Client-Side (Deserialization Path):** Instantiated by the network pipeline via the static factory method `ChangeStateInteraction.deserialize(...)`. This occurs when a corresponding packet is received from the server.

- **Scope:** Extremely short-lived. An instance of ChangeStateInteraction exists only for the brief duration of a single network event transaction. It is created, serialized, and immediately becomes eligible for garbage collection on the sender's side. On the receiver's side, it is deserialized, its data is consumed by a handler, and it is then discarded.

- **Destruction:** The object is managed by the Java garbage collector. No manual resource management is required. Systems should not hold long-term references to these objects.

## Internal State & Concurrency
- **State:** Mutable. All fields are publicly accessible or modifiable through constructors. This design is intentional for a DTO, allowing network systems to construct or populate the object efficiently during the serialization and deserialization processes.

- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, processed, and discarded within the context of a single thread, such as a Netty I/O worker or the main game thread.
    - **WARNING:** Sharing an instance of ChangeStateInteraction across threads without explicit external synchronization will lead to race conditions and unpredictable behavior, especially if one thread is modifying it while another is serializing or reading it. The network layer is responsible for ensuring a safe handoff to the game logic thread.

## API Surface
The primary contract of this class revolves around its serialization and deserialization capabilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static ChangeStateInteraction | O(N) | Constructs an object by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | int | O(N) | Writes the object's state to a ByteBuf. Returns the number of bytes written. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a structural sanity check on the byte data without full deserialization. |
| computeBytesConsumed(buf, offset) | static int | O(N) | Calculates the total size of a serialized object within a buffer. |
| computeSize() | int | O(N) | Pre-calculates the required buffer size for serialization. |

*Complexity O(N) refers to the total size of the variable-length data (maps, arrays, strings).*

## Integration Patterns

### Standard Usage
The object is almost exclusively handled by the network protocol engine. A game logic handler receives a fully-formed object, extracts the necessary data, and applies it to the game state.

```java
// Executed on a network or game thread after deserialization
public void handleInteraction(ChangeStateInteraction interaction) {
    // Retrieve the target entity from the game world
    Entity target = world.getEntityById(interaction.getTargetId());

    // Apply the state changes defined in the packet
    if (target != null && interaction.stateChanges != null) {
        for (Map.Entry<String, String> entry : interaction.stateChanges.entrySet()) {
            target.setState(entry.getKey(), entry.getValue());
        }
    }

    // The 'interaction' object is now discarded
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of ChangeStateInteraction in game state components. They represent a transient event. If the data must be preserved, copy it into a persistent game state model.
- **Client-Side Instantiation:** Clients should never create this object with `new`. It is a server-authoritative message. Creating it on the client is meaningless as there is no mechanism to send it to the server.
- **Modification After Deserialization:** Do not modify a received ChangeStateInteraction object. Treat it as immutable upon receipt. Modifying it can lead to inconsistent state if other handlers also have a reference to it.

## Data Pipeline
The flow of this data object is unidirectional, from the server's game logic to the client's game logic.

> Flow:
> Server Game Logic -> **new ChangeStateInteraction()** -> serialize() -> Netty ByteBuf -> Client Network Layer -> deserialize() -> **ChangeStateInteraction Instance** -> Client Interaction Handler -> Game State Update

