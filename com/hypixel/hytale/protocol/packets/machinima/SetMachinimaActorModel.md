---
description: Architectural reference for SetMachinimaActorModel
---

# SetMachinimaActorModel

**Package:** com.hypixel.hytale.protocol.packets.machinima
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SetMachinimaActorModel implements Packet {
```

## Architecture & Concepts
The **SetMachinimaActorModel** class is a data structure, not a service. It represents a single, specific message within the Hytale network protocol, uniquely identified by **PACKET_ID** 261. Its sole purpose is to encapsulate the data required to command a client or server to change the visual model of a specified actor within a machinima scene.

This class is a fundamental component of the network layer's serialization and deserialization pipeline. It provides a high-level, object-oriented representation of a low-level binary message. The implementation details reveal a highly optimized binary format designed for performance:

*   **Nullable Bit Field:** A single byte at the start of the packet's data block acts as a bitmask to indicate which of the nullable fields (**model**, **sceneName**, **actorName**) are present in the payload. This avoids wasting bytes for null fields.
*   **Variable Data Block:** Following a fixed-size header containing offsets, a variable-length data block holds the actual field data. This allows the packet to be compact when fields are small or null, while accommodating large data payloads like complex models.

This structure is a classic pattern in game networking, balancing protocol flexibility with wire-size efficiency.

## Lifecycle & Ownership
**SetMachinimaActorModel** is a transient object with a very short lifecycle, tied directly to a single network operation.

*   **Creation:**
    *   **Outbound (Sending):** Instantiated directly by game logic using `new SetMachinimaActorModel(...)` when a machinima system needs to transmit this command. The caller populates its fields before handing it to the network layer.
    *   **Inbound (Receiving):** Instantiated by the protocol's packet dispatcher via the static factory method **deserialize**. This occurs when a raw network buffer containing packet ID 261 is read from the TCP stream.

*   **Scope:** The object exists only for the duration of its immediate processing. It is not designed to be cached or held in long-term state.

*   **Destruction:** The object becomes eligible for garbage collection as soon as it is processed. For an outbound packet, this is after its contents are serialized into a **ByteBuf**. For an inbound packet, this is after the relevant game system has consumed its data.

## Internal State & Concurrency
*   **State:** The class is a mutable container for three nullable fields: **model**, **sceneName**, and **actorName**. Its state is defined entirely by the data it carries for serialization.

*   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization. It is designed to be created and populated on one thread and then passed to another (e.g., the Netty event loop) in a "fire-and-forget" manner.

    **WARNING:** Concurrent modification of a **SetMachinimaActorModel** instance is an error and will lead to unpredictable behavior, data corruption, or race conditions during serialization. All mutations must be completed before the object is handed off to another thread or system.

## API Surface
The public contract is focused on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static SetMachinimaActorModel | O(N) | Constructs an object by reading from a **ByteBuf**. Throws **ProtocolException** on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided **ByteBuf** according to the protocol specification. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a safe, read-only check of a buffer to ensure it contains a structurally valid packet. Does not perform a full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (261). |

*N = size of the packet payload in bytes.*

## Integration Patterns

### Standard Usage
The class is used in two primary scenarios: sending a command and receiving a command.

**Sending a Command:**
```java
// 1. Game logic creates and populates the packet
Model newActorModel = loadSomeModel();
SetMachinimaActorModel packet = new SetMachinimaActorModel(
    newActorModel,
    "IntroCutscene",
    "MainCharacter"
);

// 2. The packet is handed to the network connection to be sent
playerConnection.sendPacket(packet);
```

**Receiving and Handling a Command (Conceptual):**
```java
// In a packet handler or event listener...
public void onSetActorModel(SetMachinimaActorModel packet) {
    MachinimaScene scene = machinimaManager.getScene(packet.sceneName);
    if (scene != null) {
        MachinimaActor actor = scene.getActor(packet.actorName);
        if (actor != null) {
            // 3. The data is consumed by the relevant game system
            actor.setModel(packet.model);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
*   **State Reuse:** Do not modify a packet object after it has been passed to the network layer for sending. The serialization may happen on a different thread at a later time.
    ```java
    // BAD: The packet is modified after being queued
    SetMachinimaActorModel packet = new SetMachinimaActorModel(modelA, "scene", "actor");
    connection.sendPacket(packet);
    packet.model = modelB; // This is a race condition
    ```

*   **Ignoring Validation:** On a server, failing to call **validateStructure** on incoming data before attempting to deserialize can expose the application to denial-of-service attacks via maliciously crafted packets (e.g., incorrect length fields causing huge memory allocations).

*   **Long-Term Storage:** Do not store instances of this packet in caches or as component state. It is a transient message. If the data needs to be preserved, copy it into a dedicated state object owned by the relevant game system.

## Data Pipeline
The flow of data through this component is linear and unidirectional for any given operation.

**Outbound (Client or Server Sending):**
> Game Logic -> `new SetMachinimaActorModel()` -> Network System -> **serialize()** -> Netty ByteBuf -> TCP Stream

**Inbound (Client or Server Receiving):**
> TCP Stream -> Netty ByteBuf -> Packet Dispatcher -> **deserialize()** -> **SetMachinimaActorModel** -> Game Event Bus -> Machinima System Handler

