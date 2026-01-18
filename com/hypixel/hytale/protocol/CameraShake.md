---
description: Architectural reference for CameraShake
---

# CameraShake

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class CameraShake {
```

## Architecture & Concepts
The CameraShake class is a Data Transfer Object (DTO) operating within the Hytale network protocol layer. It is not a service or manager, but rather a pure data structure that defines the binary layout for a command sent from the server to the client to trigger a camera shake effect.

Its primary architectural function is to serve as a concrete, language-agnostic contract for serializing and deserializing camera effect parameters over the network. The structure provides distinct configurations for **firstPerson** and **thirdPerson** camera perspectives, allowing for granular control over the visual experience based on the player's current view.

The serialization format is highly optimized for network efficiency. It employs a custom binary layout consisting of a fixed-size header and a variable-size data block.

-   **Null Bit Field:** A single byte at the start of the block acts as a bitmask to indicate which of the nullable child objects (firstPerson, thirdPerson) are present in the payload. This avoids wasting network bandwidth for unused optional fields.
-   **Offset Pointers:** The fixed-size block contains integer offsets that point to the start of each variable-sized child object's data within the buffer. This allows for efficient parsing and validation without needing to traverse the entire structure sequentially.

This design pattern is common in high-performance networking to create compact, forward-compatible, and rapidly parsable data packets.

## Lifecycle & Ownership
-   **Creation:**
    -   **Sending Peer (Server):** Instantiated directly via `new CameraShake(...)` by server-side game logic when an event (e.g., an explosion) needs to trigger a camera shake for a client.
    -   **Receiving Peer (Client):** Instantiated by a protocol decoder, typically a Netty channel handler, which calls the static `deserialize` method upon receiving the relevant network packet.
-   **Scope:** Extremely short-lived and transient. An instance of CameraShake exists only for the brief moment it is being serialized into a network buffer or deserialized and processed by the client's camera rendering system.
-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been processed. There is no expectation for instances to be pooled or reused.

## Internal State & Concurrency
-   **State:** Mutable. Its fields are public and can be modified after construction. It is a simple container for data and holds no internal caches or complex state.
-   **Thread Safety:** This class is **not thread-safe**. It is designed for use within a single-threaded context, such as a Netty I/O thread or the main game loop. All serialization and deserialization operations on a shared ByteBuf must be externally synchronized. Modifying a CameraShake instance from multiple threads without explicit locking will result in undefined behavior.

## API Surface
The public API is focused exclusively on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static CameraShake | O(N) | Constructs a CameraShake object by reading from a binary buffer at a given offset. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into a binary buffer according to the defined protocol format. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Verifies that the data in a buffer represents a valid and safe-to-parse CameraShake structure. |
| computeSize() | int | O(N) | Calculates the total number of bytes required to serialize the object and its children. |
| clone() | CameraShake | O(N) | Creates a deep copy of the object and its contained CameraShakeConfig instances. |

*Complexity O(N) refers to the number of non-null child objects.*

## Integration Patterns

### Standard Usage
The class is used as part of the network packet serialization pipeline.

**Server-Side (Sending a Packet)**
```java
// 1. Create the configuration for the effect
CameraShakeConfig firstPersonConfig = new CameraShakeConfig(...);
CameraShakeConfig thirdPersonConfig = new CameraShakeConfig(...);

// 2. Instantiate the data structure
CameraShake shakeEffect = new CameraShake(firstPersonConfig, thirdPersonConfig);

// 3. Pass to a packet which will call serialize() during encoding
SomeGamePacket packet = new SomeGamePacket(shakeEffect);
channel.writeAndFlush(packet);
```

**Client-Side (Receiving a Packet)**
```java
// In a Netty ChannelInboundHandler or similar decoder:
// 1. Validate the incoming data before parsing to prevent exploits.
ValidationResult result = CameraShake.validateStructure(buffer, offset);
if (!result.isValid()) {
    throw new CorruptedDataException(result.error());
}

// 2. Deserialize the data into an object.
CameraShake receivedEffect = CameraShake.deserialize(buffer, offset);

// 3. Dispatch the object to the relevant game system.
game.getCameraSystem().applyShake(receivedEffect);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Caching:** Do not cache or reuse CameraShake instances. They are lightweight DTOs and should be created on-demand to prevent state corruption between different network messages.
-   **Ignoring Validation:** Never call `deserialize` on a buffer received from an untrusted source (i.e., any remote client or server) without first calling `validateStructure`. Skipping this step can lead to buffer over-reads, crashes, and potential security vulnerabilities.
-   **Cross-Thread Access:** Do not create an instance on the main game thread and pass it to a network thread for serialization without a proper thread-safe handoff mechanism (e.g., a concurrent queue). Direct modification from multiple threads is unsafe.

## Data Pipeline
The flow of CameraShake data is unidirectional from the game simulation to the network and from the network to the client's renderer.

**Outbound (Server)**
> Game Event -> **new CameraShake()** -> Packet Serialization -> `CameraShake.serialize(buf)` -> Netty Encoder -> Network Socket

**Inbound (Client)**
> Network Socket -> Netty Decoder -> Packet Deserialization -> `CameraShake.deserialize(buf)` -> **CameraShake Instance** -> Camera Controller -> Render Update

