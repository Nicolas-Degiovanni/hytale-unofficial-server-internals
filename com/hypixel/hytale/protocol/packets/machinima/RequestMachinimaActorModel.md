---
description: Architectural reference for RequestMachinimaActorModel
---

# RequestMachinimaActorModel

**Package:** com.hypixel.hytale.protocol.packets.machinima
**Type:** Transient

## Definition
```java
// Signature
public class RequestMachinimaActorModel implements Packet {
```

## Architecture & Concepts

The RequestMachinimaActorModel class is a Data Transfer Object (DTO) that represents a single, specific message within the Hytale network protocol. It is not a service or manager; its sole purpose is to structure data for network transmission related to the in-game Machinima creation tools.

This packet is sent by a client to a server to request a specific visual model for an actor entity within a given Machinima scene. The server is expected to respond with the requested model data or an error if the request is invalid.

Architecturally, this packet employs a highly optimized binary layout for efficiency. The structure consists of two main parts:

1.  **Fixed-Size Header (13 bytes):** This block contains metadata about the payload.
    *   **Null Bit Field (1 byte):** A bitmask indicating which of the nullable string fields are present in the payload. This avoids wasting bytes for null values.
    *   **Offset Pointers (3 x 4 bytes):** Three little-endian integers, each pointing to the starting position of a variable-length string within the data block. This allows for direct-access parsing and validation without reading the entire packet sequentially.

2.  **Variable-Size Data Block:** This block contains the actual string data for `modelId`, `sceneName`, and `actorName`. The strings are prefixed with a VarInt to encode their length.

This design prioritizes performance and security by enabling rapid validation of offsets and lengths before committing to full deserialization, mitigating certain classes of buffer overflow attacks.

### Lifecycle & Ownership

-   **Creation:**
    -   **Outbound (Client-Side):** Instantiated by game logic, typically in response to a user action within the Machinima editor. The fields are populated, and the object is passed to the network layer for serialization.
    -   **Inbound (Server-Side):** Instantiated by the protocol's packet factory via the static `deserialize` method when a raw network buffer with Packet ID 260 is received.

-   **Scope:** This object is extremely short-lived and scoped to a single network transaction. It is created, used for either serialization or processing by a handler, and then immediately becomes eligible for garbage collection.

-   **Destruction:** There is no manual destruction logic. The Java Garbage Collector reclaims the memory once the network pipeline or packet handler releases its reference. **WARNING:** Holding long-term references to packet objects is a memory leak anti-pattern.

## Internal State & Concurrency

-   **State:** The class is a mutable data container. Its state consists of three nullable String fields. It does not cache data or maintain connections to other systems. Once deserialized or before serialization, its state is considered fixed for the duration of its brief lifecycle.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and serialized within a single thread, or deserialized and processed within a single network I/O thread (e.g., a Netty event loop). Any handoff to other threads (like the main game thread) must be done through thread-safe mechanisms like a concurrent queue. **WARNING:** Concurrent modification of a packet's fields will lead to race conditions and unpredictable serialization output.

## API Surface

The primary contract of this class revolves around its serialization and deserialization capabilities.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided Netty ByteBuf according to the defined binary protocol. N is the total length of all strings. |
| deserialize(ByteBuf, int) | RequestMachinimaActorModel | O(N) | Static factory method. Decodes a new instance from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| computeSize() | int | O(N) | Calculates the total number of bytes required to serialize the object. |
| validateStructure(ByteBuf, int) | ValidationResult | O(1) | Performs cheap, non-allocating checks on a buffer to validate offsets and lengths without full deserialization. Critical for network security. |

## Integration Patterns

### Standard Usage

Developers typically do not interact with the serialization or deserialization methods directly. The network engine handles this. The primary interaction is either creating a packet to be sent or handling a received packet.

**Sending a Packet (Client-Side Logic)**
```java
// 1. Create and populate the packet
RequestMachinimaActorModel request = new RequestMachinimaActorModel(
    "hytale:player_male", 
    "intro_sequence", 
    "main_character_1"
);

// 2. Pass to the network system to be sent
// (This is a conceptual example)
clientConnection.sendPacket(request);
```

**Handling a Packet (Server-Side Logic)**
A packet handler or listener receives the fully-formed object.
```java
// In a packet handler class...
public void handle(RequestMachinimaActorModel packet) {
    String model = packet.modelId;
    String scene = packet.sceneName;
    
    // Process the request...
    MachinimaService service = context.getService(MachinimaService.class);
    service.loadActorModel(model, scene, packet.actorName);
}
```

### Anti-Patterns (Do NOT do this)

-   **Object Re-use:** Do not modify and re-send the same packet instance. Packets are cheap to create and should be treated as single-use messages.
-   **Manual Deserialization:** Never call `deserialize` directly unless you are implementing a custom network pipeline. The engine's protocol decoder is responsible for this.
-   **Stateful Packets:** Do not add complex logic, state, or service dependencies to a packet class. They are meant to be simple, inert data structures.

## Data Pipeline

The flow of this object through the system is linear and unidirectional for a single transaction.

**Outbound Flow (Client to Server):**
> Game Logic -> `new RequestMachinimaActorModel()` -> Network Pipeline -> **serialize()** -> Raw ByteBuf -> TCP Socket

**Inbound Flow (Server Receives from Client):**
> TCP Socket -> Raw ByteBuf -> Protocol Decoder (identifies Packet ID 260) -> **deserialize()** -> `RequestMachinimaActorModel` Instance -> Packet Handler -> Game Logic

