---
description: Architectural reference for UpdateVisibleHudComponents
---

# UpdateVisibleHudComponents

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class UpdateVisibleHudComponents implements Packet {
```

## Architecture & Concepts
The UpdateVisibleHudComponents class is a network protocol Data Transfer Object, not a service or manager. Its sole responsibility is to represent a message sent from the server to the client, instructing the client which core Heads-Up Display (HUD) elements should be rendered.

As an implementation of the **Packet** interface, this class is a fundamental component of the Hytale network layer. It encapsulates the data and the serialization/deserialization logic required to transmit UI state changes over the network. The design pattern, featuring static factory methods like **deserialize** and instance methods like **serialize**, is common across the entire protocol. This allows the network dispatcher to decode a raw byte stream into a concrete packet object without prior knowledge of its specific type, using the packet ID (230) as a key.

This class is a pure data container. It contains no game logic and should be considered an immutable message once created or received.

## Lifecycle & Ownership
- **Creation:**
    - **Sending Peer (Server):** Instantiated directly via `new UpdateVisibleHudComponents(components)` by server-side game logic responsible for managing player UI state.
    - **Receiving Peer (Client):** Instantiated by the client's network protocol dispatcher. The static **deserialize** method is invoked as a factory when an incoming network message with packet ID 230 is identified in the byte stream.

- **Scope:** Transient and extremely short-lived. An instance of this class exists only for the duration of its processing within a single network event or game tick. It is a message, not a persistent state container.

- **Destruction:** The object becomes eligible for garbage collection immediately after the network event handler that received it has finished execution. There is no manual resource management or destruction required.

## Internal State & Concurrency
- **State:** Mutable. The primary state, the **visibleComponents** array, is a public field. However, by convention and design, instances should be treated as immutable after creation or deserialization. Modifying a packet after it has been queued for sending or received by a handler is a severe anti-pattern.

- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or the main game thread. All fields are accessed directly without locks or other synchronization primitives. Passing an instance between threads requires external synchronization, though the recommended pattern is to extract the data into a thread-safe structure instead.

## API Surface
The public API is dominated by static methods for protocol handling and instance methods for serialization, conforming to the **Packet** interface contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static UpdateVisibleHudComponents | O(N) | Constructs a packet from a raw network buffer. Throws ProtocolException on malformed data. N is the number of components. |
| serialize(buf) | void | O(N) | Encodes the packet's state into a network buffer for transmission. N is the number of components. |
| computeSize() | int | O(1) | Calculates the final byte size of the serialized packet without performing the full serialization operation. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a lightweight, non-deserializing check on a buffer to determine if it could contain a valid packet of this type. |
| getId() | int | O(1) | Returns the static network identifier (230) for this packet type. |

## Integration Patterns

### Standard Usage
This packet is typically created by server logic and consumed by a client-side packet handler, which then delegates the update to a dedicated UI management system. Direct interaction is rare outside of the core network and UI systems.

```java
// Example: Server-side logic to send the packet
HudComponent[] componentsToShow = { HudComponent.HEALTH_BAR, HudComponent.CROSSHAIR };
UpdateVisibleHudComponents packet = new UpdateVisibleHudComponents(componentsToShow);

// The network connection manager handles the actual serialization and sending
playerConnection.sendPacket(packet);

// Example: Client-side handler (conceptual)
public void handlePacket(UpdateVisibleHudComponents packet) {
    // Delegate the data to the UI system, which knows how to render components
    this.uiManager.setVisibleHudElements(packet.visibleComponents);
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not hold a reference to a packet instance and modify its contents for subsequent network messages. Packets are designed to be cheap, single-use objects. Create a new instance for every message to ensure state integrity.
- **Cross-Thread Modification:** Do not deserialize a packet on a network thread and then pass its reference to the main game thread for modification. This is a classic race condition. Treat the received packet as an immutable message and copy its data if modifications are needed in another thread.
- **Manual Buffer Management:** Avoid calling **serialize** directly. The network layer abstracts this away. Rely on higher-level APIs like a `Connection.sendPacket` method to manage the underlying ByteBuf lifecycle.

## Data Pipeline
This packet follows a simple, unidirectional flow from the game server to the game client.

> Flow:
> Server Game Logic -> **new UpdateVisibleHudComponents()** -> Server Network Layer (Serialization) -> TCP/IP Stack -> Client Network Layer (Deserialization) -> Client Packet Handler -> UI System Update

