---
description: Architectural reference for UpdateKnownRecipes
---

# UpdateKnownRecipes

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class UpdateKnownRecipes implements Packet {
```

## Architecture & Concepts
The UpdateKnownRecipes class is a network protocol Data Transfer Object (DTO). It serves as a data contract, defining the structure for synchronizing a player's known crafting recipes between the server and the client. This class is not a service or manager; it is a pure data container, a message sent from the server to a client.

Its primary role is to encapsulate a complete or partial list of recipes a player has unlocked. The client's user interface systems, such as the crafting menu, consume the data from this packet to dynamically render the available crafting options.

The design emphasizes network efficiency. It employs a custom binary serialization format that utilizes a nullability bitfield and variable-length integers (VarInt) to minimize the packet's size on the wire. This is critical for performance, especially when large recipe lists are transmitted, for instance, upon a player's initial login.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side (Sending):** Instantiated by game logic when a player's recipe book must be updated. This typically occurs on login, after a player discovers a new recipe, or when an administrator grants recipes. The server populates the *known* map and hands the object to the network layer for serialization.
    - **Client-Side (Receiving):** Instantiated within the network protocol decoder, typically a Netty pipeline handler. The static factory method *deserialize* is called to construct the object from an incoming ByteBuf when a packet with ID 228 is detected.

- **Scope:** Transient. An instance of UpdateKnownRecipes has an extremely short lifespan. It exists only for the brief period between deserialization and processing by a client-side system, or between creation and serialization on the server-side.

- **Destruction:** The object is immediately eligible for garbage collection after its data has been consumed by the relevant handler or service. No long-term references are maintained.

## Internal State & Concurrency
- **State:** Mutable. The core state is the public *known* map, which can be directly modified. However, by convention, the object is treated as immutable after its initial population (on the server) or deserialization (on the client).

- **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous processing within a single network or game thread. All serialization, deserialization, and data processing must occur in a thread-safe context, typically managed by the Netty event loop or the main game thread. Unsynchronized access from multiple threads will result in data corruption and undefined behavior.

## API Surface
The public API is dominated by static methods for serialization and validation, reflecting its role as a DTO within a protocol framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static UpdateKnownRecipes | O(N) | Constructs an object from a binary representation in a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. Throws ProtocolException if constraints are violated. |
| computeSize() | int | O(M) | Calculates the byte size of the serialized packet without performing serialization. M is the number of recipes. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check on a ByteBuf to verify if it contains a structurally valid packet. Does not allocate memory. |

*N = Number of bytes in the serialized map data. M = Number of entries in the recipe map.*

## Integration Patterns

### Standard Usage
The primary integration pattern involves a network handler decoding the packet and dispatching its data to a dedicated service for state management.

**Client-Side Processing Example:**
```java
// In a network message handler...
UpdateKnownRecipes packet = UpdateKnownRecipes.deserialize(buffer, offset);

// Dispatch the data to a service that manages player state.
// This decouples network logic from game logic.
PlayerStateService playerState = context.getService(PlayerStateService.class);
playerState.updateKnownRecipes(packet.known);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the *known* map after the packet has been received and is being processed by game systems. Treat it as a read-only structure to prevent inconsistent state.
- **Client-Side Instantiation:** Clients should never create this packet using *new UpdateKnownRecipes()*. It is a server-authoritative message. Attempting to create and send this packet from a client would be a protocol violation.
- **Shared References:** Do not hold a reference to this packet object after its initial processing. Copy the data into the client's own data structures and allow the packet object to be garbage collected.

## Data Pipeline
The flow of data encapsulated by this packet is unidirectional from server to client.

> **Flow:**
> Server Game Logic → **UpdateKnownRecipes (Instance)** → Network Encoder → Serialized ByteBuf → Client Network Decoder → **UpdateKnownRecipes (Instance)** → Client Game Service → UI State Update

