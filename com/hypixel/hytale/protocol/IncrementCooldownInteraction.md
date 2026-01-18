---
description: Architectural reference for IncrementCooldownInteraction
---

# IncrementCooldownInteraction

**Package:** com.hypixel.hytale.protocol
**Type:** Transient (Data Transfer Object)

## Definition
```java
// Signature
public class IncrementCooldownInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The IncrementCooldownInteraction class is a specialized Data Transfer Object (DTO) within the Hytale network protocol layer. It represents a specific, concrete game event: the modification of a cooldown timer for a player ability or item. As a subclass of SimpleInteraction, it is part of a larger hierarchy of game actions that can be serialized and transmitted between the client and server.

This class is not a service or manager; it is pure data. Its primary role is to encapsulate the state required to describe a cooldown change. The presence of highly optimized, manual serialization and deserialization methods (e.g., `serialize`, `deserialize`) indicates its use in a performance-critical binary protocol. The structure of the serialization format, which uses a fixed-size block followed by a variable-size block for nullable or complex types, is a common pattern for minimizing network payload size. This class is a fundamental building block for the game's interaction system, acting as a structured message that communicates state changes.

## Lifecycle & Ownership
- **Creation:** An instance is created under two primary circumstances:
    1. **Deserialization:** The static `deserialize` method constructs an object from an incoming network ByteBuf. This is the most common creation path on the receiving end (client or server).
    2. **Direct Instantiation:** Game logic on the sending end creates an instance via its constructor to represent a new cooldown event that needs to be transmitted.
- **Scope:** The object's lifetime is extremely short and transactional. It exists only to be serialized into a buffer for network transmission or to be read from a buffer and processed by a handler. It is not designed to be stored or referenced long-term.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it has been processed by the relevant game logic or network handler and no more strong references to it exist.

## Internal State & Concurrency
- **State:** The state is entirely mutable. All fields are public and directly accessible, which is characteristic of a DTO designed for high-performance serialization. The class holds no cached data and its state directly reflects the data of a single, discrete cooldown event.
- **Thread Safety:** This class is **not thread-safe**. Its public, mutable fields and lack of internal synchronization mechanisms make it inherently unsafe for concurrent access. Instances of IncrementCooldownInteraction must only be created, modified, and read from a single thread, typically a Netty I/O thread or the main game logic thread.

**WARNING:** Sharing instances of this class across threads without explicit, external locking will lead to race conditions, data corruption, and unpredictable behavior.

## API Surface
The public API is dominated by methods related to network serialization and object state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | IncrementCooldownInteraction | O(N) | **[Static]** Constructs an object by reading from a binary buffer. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into a binary buffer. Returns the number of bytes written. |
| computeSize() | int | O(N) | Calculates the total byte size the object will occupy when serialized. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | **[Static]** Performs a read-only check of a buffer to ensure it contains a valid representation of this object. |
| clone() | IncrementCooldownInteraction | O(N) | Creates a deep copy of the object and its contained data structures. |

*N = size of the variable-length data fields (e.g., strings, maps, arrays).*

## Integration Patterns

### Standard Usage
The class is intended to be used as part of the network protocol pipeline. Game logic creates and populates the object, then hands it off to a network system for serialization and transmission.

```java
// Example: Server-side logic sending a cooldown update to a client
IncrementCooldownInteraction interaction = new IncrementCooldownInteraction();

// Populate the required fields
interaction.cooldownId = "primary_attack";
interaction.cooldownIncrementTime = 0.5f; // Add 0.5 seconds to the cooldown
interaction.cooldownIncrementInterrupt = false;

// ... set other inherited fields from SimpleInteraction

// Pass the object to the network layer to be sent to the client
playerConnection.sendPacket(interaction);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not hold references to this object in long-lived components. It is a transient message, not a state container. Caching or reusing instances is unsafe and can lead to bugs where old data is accidentally resent.
- **Concurrent Modification:** Never modify an instance from one thread while another thread is serializing or reading it. This will corrupt the network stream.
- **Manual Buffer Management:** Avoid calling `serialize` directly into a manually managed buffer. The higher-level network protocol manager is responsible for packet framing, length prefixing, and buffer lifecycle.

## Data Pipeline
IncrementCooldownInteraction serves as a data payload within the network stack. Its flow is linear and unidirectional for any single event.

> **Outbound Flow:**
> Game Logic -> **new IncrementCooldownInteraction()** -> Network Encoder -> `serialize()` -> Netty ByteBuf -> TCP/UDP Socket

> **Inbound Flow:**
> TCP/UDP Socket -> Netty ByteBuf -> Protocol Frame Decoder -> `deserialize()` -> **IncrementCooldownInteraction instance** -> Game Event Handler

