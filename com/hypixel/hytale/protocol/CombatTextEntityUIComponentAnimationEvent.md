---
description: Architectural reference for CombatTextEntityUIComponentAnimationEvent
---

# CombatTextEntityUIComponentAnimationEvent

**Package:** com.hypixel.hytale.protocol
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class CombatTextEntityUIComponentAnimationEvent {
```

## Architecture & Concepts
The CombatTextEntityUIComponentAnimationEvent is a specialized, high-performance Data Transfer Object designed for network serialization. It does not contain any game logic. Its sole purpose is to represent the state of a single, discrete animation event for a client-side UI component, such as the floating text that appears during combat to indicate damage or healing.

This class is a fundamental building block of the client-server communication protocol. It is engineered for minimal overhead, featuring a fixed-size memory layout (34 bytes) and direct serialization/deserialization to and from Netty's ByteBuf. This design avoids the performance penalties associated with reflection-based serialization frameworks, which is critical for the high-frequency updates required in a real-time game environment.

It acts as a data contract between the server's combat simulation and the client's rendering engine. The server originates these events, and the client consumes them to drive visual effects.

## Lifecycle & Ownership
- **Creation:**
    - **Server-Side:** Instantiated by the game's combat logic systems when an action requiring a visual indicator occurs (e.g., a player deals damage). The new object is populated with animation parameters and immediately serialized into a network packet.
    - **Client-Side:** Instantiated exclusively by the network protocol layer via the static `deserialize` factory method when an incoming packet is decoded.

- **Scope:** This object is **transient** and has an extremely short lifespan. It is designed to be created, transmitted, and consumed within a single update tick.

- **Destruction:** The object becomes eligible for garbage collection immediately after its data is processed. On the server, this occurs after serialization. On the client, this occurs after the UI animation system has consumed its parameters. There should be no long-term references to instances of this class.

## Internal State & Concurrency
- **State:** The object's state is fully **mutable** via public fields. This is an intentional design choice for a DTO, allowing for efficient, direct population of its data before serialization without the overhead of setters. The state includes all necessary parameters to define a simple animation curve: start/end times, scales, opacities, and an optional position offset.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit, external synchronization. It is intended for use within a single-threaded context, such as a server's main game loop or a client's Netty event loop. Unsynchronized multi-threaded access will lead to data corruption during serialization or consumption.

## API Surface
The public contract is dominated by serialization and deserialization logic. Standard object methods like `equals` and `hashCode` are present but are secondary to the primary networking functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static CombatTextEntityUIComponentAnimationEvent | O(1) | Constructs a new instance by reading a fixed 34-byte block from the provided buffer at a given offset. This is the primary entry point on the client. |
| serialize(ByteBuf) | void | O(1) | Writes the object's state into a fixed 34-byte block in the provided buffer. This is the primary exit point on the server. |
| clone() | CombatTextEntityUIComponentAnimationEvent | O(1) | Creates a shallow copy of the object, cloning the nested Vector2f if present. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Performs a pre-check to ensure the buffer contains enough readable bytes for a successful deserialization. |

## Integration Patterns

### Standard Usage
The object is used as part of a fire-and-forget network event pipeline.

**Server-Side Generation:**
```java
// Executed within the server's game logic
CombatTextEntityUIComponentAnimationEvent event = new CombatTextEntityUIComponentAnimationEvent();
event.type = CombatTextEntityUIAnimationEventType.Scale;
event.startAt = 0.0f;
event.endAt = 0.5f;
// ... populate other fields

// The network layer serializes the event into a packet buffer
packetBuilder.addEvent(event);
```

**Client-Side Consumption:**
```java
// Executed within the client's network message handler
// The 'buffer' and 'offset' are provided by the protocol decoder
CombatTextEntityUIComponentAnimationEvent event = CombatTextEntityUIComponentAnimationEvent.deserialize(buffer, offset);

// The event is then passed to the responsible UI system
uiAnimationManager.playAnimation(event);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not cache or reuse instances of this class. Each event is a unique, point-in-time message. Reusing an object can lead to sending or rendering stale animation data.

- **Client-Side Instantiation:** Do not use `new CombatTextEntityUIComponentAnimationEvent()` on the client. Client-side instances must only originate from the `deserialize` method to ensure they reflect authoritative server state.

- **Modification After Deserialization:** Do not modify the state of an event object on the client after it has been deserialized. It should be treated as an immutable record once received.

## Data Pipeline
The flow of this data object is unidirectional, from server simulation to client presentation.

> **Flow:**
> Server Combat Logic → **new CombatTextEntityUIComponentAnimationEvent()** → Serialization → Network Packet → Client Packet Decoder → **CombatTextEntityUIComponentAnimationEvent.deserialize()** → UI Animation System → Render Engine

