---
description: Architectural reference for PlayInteractionFor
---

# PlayInteractionFor

**Package:** com.hypixel.hytale.protocol.packets.interaction
**Type:** Transient

## Definition
```java
// Signature
public class PlayInteractionFor implements Packet {
```

## Architecture & Concepts
The PlayInteractionFor class is a Data Transfer Object (DTO) that represents a specific client-to-server network message. It serves as the structured data contract for when a player's client informs the server that it is attempting to perform an interaction with an entity or object in the game world.

This packet is a fundamental component of the client-authoritative input model for interactions. The client initiates the action, bundles all necessary context—such as the target entity, the type of interaction (primary/secondary), and any associated item—into this packet, and transmits it to the server. The server then receives this packet, validates the request against game state and rules, and executes the interaction's logic if it is deemed valid.

This class is not a service or manager; it contains no business logic. Its sole responsibility is to encapsulate the data for an interaction event for network transport. Its structure is tightly coupled to the binary wire format, with methods dedicated to high-performance serialization and deserialization using Netty's ByteBuf.

## Lifecycle & Ownership
-   **Creation:** An instance of PlayInteractionFor is created under two distinct circumstances:
    1.  **Client-Side (Sending):** Instantiated by the client's input handling or player controller systems when a user performs an action (e.g., left-clicking on an entity). The new object is populated with data and passed to the network layer for serialization.
    2.  **Server-Side (Receiving):** Instantiated by the server's network protocol layer. When an incoming data stream is identified by PACKET_ID 292, the static deserialize method is invoked to construct a PlayInteractionFor object from the raw byte buffer.

-   **Scope:** The object's lifetime is extremely short and bound to a single transaction. It exists only to be passed from the point of creation to the point of processing. On the client, it is eligible for garbage collection after being serialized to the network buffer. On the server, it is eligible for garbage collection after its data has been consumed by the relevant game logic handler.

-   **Destruction:** There is no explicit destruction method. The object is managed by the Java garbage collector and is reclaimed once it falls out of the scope of the network or game logic thread that was processing it.

## Internal State & Concurrency
-   **State:** The class is fully **mutable**, with all fields exposed as public members. This design choice prioritizes performance by avoiding method call overhead for getters and setters during the high-frequency operations of serialization and deserialization.

-   **Thread Safety:** **This class is not thread-safe.** It is a simple data container with no internal locking or synchronization mechanisms. It is designed to be created, populated, and processed within a single thread, typically a Netty I/O thread or a dedicated game loop thread.

    **WARNING:** Sharing an instance of PlayInteractionFor across multiple threads without external, explicit synchronization will lead to race conditions and unpredictable behavior. It must be treated as thread-confined.

## API Surface
The primary contract of this class is its static methods for protocol handling, not instance methods for game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static PlayInteractionFor | O(N) | Constructs a new packet instance from a binary buffer. N is the size of variable-length fields. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the packet's state into the provided binary buffer. N is the size of variable-length fields. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a non-deserializing check on the buffer to validate offsets and lengths. Critical for security and preventing malformed packet exploits. |
| computeSize() | int | O(N) | Calculates the total byte size required to serialize the current state of the packet. Used for buffer pre-allocation. |
| getId() | int | O(1) | Returns the static network identifier for this packet type (292). Part of the Packet interface contract. |

## Integration Patterns

### Standard Usage
This packet is intended to be processed by a server-side network handler. The handler deserializes the packet and then dispatches it to the appropriate game system for processing, often via an event bus or direct service call.

```java
// Example of a server-side packet handler
public class InteractionPacketHandler {

    private final GameLogicService gameLogic;

    public void handle(PlayInteractionFor packet) {
        // The packet is received fully formed from the network layer.
        // Do not modify the packet here. Read its data and pass to services.
        
        Entity target = world.getEntity(packet.entityId);
        if (target != null) {
            gameLogic.processPlayerInteraction(
                packet.entityId,
                packet.interactionType,
                packet.interactedItemId
            );
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **State Holding:** Do not retain instances of PlayInteractionFor in long-lived components. It is a transient message, not a state object. Copy its data into your domain models if you need to persist it.
-   **Cross-Thread Modification:** Do not create a packet on one thread and populate its fields on another. All modifications must happen before it is handed off for serialization or processing.
-   **Reusing Instances:** Do not attempt to reuse a packet instance for multiple network messages. Always create a new instance for each distinct interaction event to ensure data integrity.

## Data Pipeline
The flow of data encapsulated by this packet travels from the client's input hardware to the server's game logic.

> Flow (Client to Server):
> User Input (Mouse Click) -> Client Input System -> New **PlayInteractionFor** instance created -> Network Encoder calls **serialize()** -> TCP Stream -> Server Network Layer -> Packet Dispatcher calls **deserialize()** -> Server-Side Packet Handler -> Game Logic System

