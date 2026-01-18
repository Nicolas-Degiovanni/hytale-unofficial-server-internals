---
description: Architectural reference for MovementStates
---

# MovementStates

**Package:** com.hypixel.hytale.protocol
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class MovementStates {
```

## Architecture & Concepts
MovementStates is a fundamental data transfer object (DTO) within the Hytale network protocol. It is not a service or manager, but a plain data structure designed for high-performance serialization and deserialization of an entity's physical state.

Its primary role is to serve as a network contract, encapsulating a comprehensive snapshot of 22 distinct boolean flags that define how an entity is moving or interacting with the world at a specific moment. This includes states like *jumping*, *sprinting*, *climbing*, and *inFluid*.

The class is architected with a fixed-size memory layout of 22 bytes. Each boolean state is intentionally mapped to a full byte (0 or 1) rather than being packed into a bitfield. While this increases the payload size, it provides a significant performance advantage by eliminating the need for bitwise masking and shifting operations during serialization. This design choice prioritizes raw processing speed on both the client and server, which is critical for high-frequency state updates.

MovementStates is a core component of larger network packets, typically embedded within player update or entity synchronization messages.

## Lifecycle & Ownership
- **Creation:** Instances are created ephemerally under two primary conditions:
    1.  **Deserialization:** The static `deserialize` factory method is invoked by the network protocol decoder when an incoming packet containing movement data is being parsed.
    2.  **State Capture:** The game client's physics or entity-control system instantiates a MovementStates object on each relevant tick to capture the player's current state before serializing it for transmission to the server.
- **Scope:** The lifetime of a MovementStates object is extremely short. It is designed to be transient and is scoped to the processing of a single network packet or a single game tick.
- **Destruction:** The object is managed by the Java garbage collector and becomes eligible for collection as soon as all references to it are dropped, which typically occurs immediately after a network packet has been fully processed or sent. There are no manual cleanup requirements.

## Internal State & Concurrency
- **State:** The object is a mutable container of boolean flags. All state-bearing fields are public, allowing for direct and efficient modification. It does not cache any data and purely represents the state passed to it.
- **Thread Safety:** **This class is not thread-safe.** Its public mutable fields make it inherently unsafe for concurrent access. It is designed to be created, populated, and consumed within the confines of a single thread, such as a Netty I/O thread or the main game loop thread.

**WARNING:** Sharing a MovementStates instance across threads without external locking mechanisms will lead to race conditions, packet corruption, and unpredictable game state. Such a pattern is a severe anti-pattern.

## API Surface
The public API is focused exclusively on serialization, state transfer, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static MovementStates | O(1) | Constructs a new MovementStates object by reading 22 bytes from the provided buffer at a given offset. |
| serialize(ByteBuf) | void | O(1) | Writes the 22 boolean fields as 22 bytes into the provided buffer. |
| computeSize() | int | O(1) | Returns the constant size of the serialized object, which is always 22. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes (22) to deserialize the object. |
| clone() | MovementStates | O(1) | Creates a new MovementStates instance with a copy of the current object's state. |

## Integration Patterns

### Standard Usage
MovementStates is almost never used in isolation. It is intended to be a component of a larger network packet. The parent packet is responsible for invoking the serialization and deserialization logic.

```java
// Example: Within a hypothetical PlayerUpdatePacket's deserialization logic
public void deserialize(ByteBuf packetBuffer) {
    // ... deserialize other packet fields ...
    int movementStateOffset = ...; // Calculate offset from packet start
    this.movement = MovementStates.deserialize(packetBuffer, movementStateOffset);
    // ... continue deserializing ...
}

// Example: Capturing and sending client state
MovementStates currentState = new MovementStates();
currentState.onGround = player.isOnGround();
currentState.sprinting = player.isSprinting();
// ... populate all 22 fields ...

PlayerUpdatePacket packet = new PlayerUpdatePacket(currentState);
connection.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **Long-Term Storage:** Do not retain instances of MovementStates as part of a persistent entity state. It represents a point-in-time snapshot and will quickly become stale. Always generate a fresh instance for new updates.
- **Concurrent Access:** As stated previously, do not read a MovementStates instance from one thread while another thread is writing to it. This will cause severe data integrity issues.
- **Partial Population:** When creating an instance for serialization, ensure all 22 fields are explicitly set. Relying on default `false` values can lead to logical errors if a state is intended to be true but is missed.

## Data Pipeline
MovementStates acts as a data marshalling structure between the game logic and the raw network byte stream.

**Client to Server (Outbound Flow):**
> Player Input -> Client Physics Tick -> **MovementStates (new instance)** -> Parent Packet Serialization -> **MovementStates.serialize()** -> Netty ByteBuf -> Network Socket

**Server from Client (Inbound Flow):**
> Network Socket -> Netty ByteBuf -> Parent Packet Deserialization -> **MovementStates.deserialize()** -> **MovementStates (new instance)** -> Server Game Logic -> Entity State Update

