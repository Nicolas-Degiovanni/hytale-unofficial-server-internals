---
description: Architectural reference for ChainingInteraction
---

# ChainingInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class ChainingInteraction extends Interaction {
```

## Architecture & Concepts
The **ChainingInteraction** class is a Data Transfer Object (DTO) that defines the wire format for a specific, complex type of player or entity interaction within the Hytale protocol layer. It extends the base **Interaction** class, inheriting common properties while adding fields specific to sequencing or "chaining" multiple interactions together, such as for attack combos or multi-stage abilities.

Its primary architectural role is to serve as a concrete, language-native representation of a network message. The class encapsulates the complex logic for serialization and deserialization between its object state and a raw Netty **ByteBuf**.

The binary layout is highly optimized for network performance and consists of two main parts:
1.  **Fixed-Size Block:** A 47-byte header containing primitive types, fixed-size fields, and a crucial one-byte bitfield named *nullBits*. This bitfield indicates which of the subsequent variable-size fields are present in the payload.
2.  **Variable-Size Block:** A data region following the fixed block. The fixed block contains 32-bit integer offsets pointing to the start of each data structure within this variable block. This design avoids parsing the entire message to access a specific field and efficiently handles optional data.

This class is a fundamental building block of the client-server communication contract. Any changes to its structure or serialization logic are breaking changes to the network protocol.

### Lifecycle & Ownership
- **Creation:** An instance of **ChainingInteraction** is created under two circumstances:
    1.  **Inbound:** The static **deserialize** method is invoked by a network packet decoder when an incoming **ByteBuf** is identified as a chaining interaction message. The network layer owns this creation process.
    2.  **Outbound:** Game logic on the server or client instantiates the class via its constructor to build an interaction that will be sent over the network.
- **Scope:** This object is ephemeral and designed for a very short lifecycle. It should only exist for the duration of a single network event processing tick. It is a message, not a persistent state container.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after the relevant game system has processed the interaction data.

## Internal State & Concurrency
- **State:** The state of a **ChainingInteraction** instance is fully **mutable**. All data fields are public, allowing for direct modification after instantiation. This design prioritizes performance by avoiding getter/setter overhead, with the expectation that the object is populated once and then treated as read-only.

- **Thread Safety:** **This class is not thread-safe.** Its public mutable fields and lack of any synchronization mechanisms make it inherently unsafe for concurrent access from multiple threads.

    **WARNING:** Instances of **ChainingInteraction** must be confined to a single thread, typically the Netty event loop thread that deserialized it or the main game thread that created it. Do not share instances across threads. If multi-threaded access is required, the receiving thread must create a deep copy.

## API Surface
The public contract is dominated by static methods for serialization, deserialization, and validation from a **ByteBuf**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | ChainingInteraction | O(N) | **[Static]** Constructs a new object by reading from a buffer at a given offset. Throws **ProtocolException** on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided buffer. Returns the number of bytes written. |
| computeBytesConsumed(ByteBuf, int) | int | O(N) | **[Static]** Calculates the total size of a serialized object within a buffer without full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes this object would occupy if serialized. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **[Static]** Performs a structural integrity check on the binary data in a buffer without creating an object. Crucial for security and preventing parsing exploits. |
| clone() | ChainingInteraction | O(N) | Creates a copy of the object. Note that this is not a fully deep copy; primitive collections are copied, but some object references may be shared. |

## Integration Patterns

### Standard Usage
This object is almost exclusively handled within the network layer or by game systems that directly process player input and actions. A typical use case involves a packet handler decoding the message and dispatching it.

```java
// Executed on a Netty event loop or game thread
void handleInteractionPacket(ByteBuf packetPayload) {
    // The payload is validated and deserialized into a usable object
    ChainingInteraction interaction = ChainingInteraction.deserialize(packetPayload, 0);

    // The interaction data is passed to a dedicated system for processing
    // The 'interaction' object should not be stored long-term
    game.getInteractionSystem().process(interaction);
}
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not store instances of **ChainingInteraction** in caches or as component state. They represent a point-in-time event, not persistent data. Convert its data into a stable internal state object if necessary.
- **Cross-Thread Modification:** Never modify an instance from a different thread than the one that created it. This will lead to race conditions and unpredictable behavior.
- **Ignoring Validation:** Bypassing **validateStructure** on untrusted input can expose the server to denial-of-service attacks via malformed packets (e.g., invalid length fields causing excessive memory allocation).

## Data Pipeline
The **ChainingInteraction** class is a data payload that flows through the network stack.

**Inbound Flow (Client to Server or vice-versa):**
> Network ByteBuf -> Protocol Frame Decoder -> Packet ID Dispatcher -> **ChainingInteraction.deserialize** -> Game Logic (e.g., AnimationSystem, CombatSystem)

**Outbound Flow (Server to Client or vice-versa):**
> Game Logic creates **ChainingInteraction** instance -> Packet Encoder calls **interaction.serialize** -> Protocol Frame Assembler -> Network ByteBuf

