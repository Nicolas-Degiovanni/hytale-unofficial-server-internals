---
description: Architectural reference for HideEventTitle
---

# HideEventTitle

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient

## Definition
```java
// Signature
public class HideEventTitle implements Packet {
```

## Architecture & Concepts
The HideEventTitle class is a Data Transfer Object (DTO) representing a specific network command within Hytale's protocol layer. Its sole purpose is to instruct a game client to hide the currently displayed event title UI element, often with a specified fade-out animation duration.

As a Packet, this class does not contain any logic. It is a pure data container, a message sent from the server to the client. It is part of a family of packets in the *interface_* package, which collectively manage the state and appearance of the player's user interface. Its design emphasizes performance and predictability, with fixed sizing and direct, low-level serialization/deserialization methods that operate on Netty's ByteBuf.

This object is a fundamental building block of the server-authoritative UI system. The server dictates UI state changes, and clients react by processing these simple, explicit command packets.

## Lifecycle & Ownership
- **Creation:**
    - **Outbound (Server):** Instantiated directly by server-side game logic when a condition is met to hide a UI title (e.g., a world event ends). The server creates an instance with the desired fade duration and queues it for sending to a client.
    - **Inbound (Client):** Instantiated by the client's network pipeline. A decoder identifies the packet ID (215) from the incoming byte stream and invokes the static *deserialize* factory method to construct the object from the raw buffer.
- **Scope:** Per-packet processing. The object's lifetime is exceptionally short, existing only for the duration of its journey through the network stack and its subsequent handling by a single event listener.
- **Destruction:** The object becomes eligible for garbage collection immediately after the client-side packet handler has consumed its data (the *fadeOutDuration*). There is no persistent state or external ownership.

## Internal State & Concurrency
- **State:** Mutable. The class holds a single public float field, *fadeOutDuration*. While technically mutable, instances should be treated as immutable after creation. Modifying a packet after it has been queued for sending or received from the network is a critical anti-pattern.
- **Thread Safety:** This class is **not thread-safe**. Packet objects are designed to be processed by a single thread, typically a Netty IO worker or the main game thread. Sharing packet instances across threads will lead to race conditions and unpredictable behavior. The protocol framework guarantees that a single packet instance is handled by only one thread.

## API Surface
The public contract is designed for interaction with the protocol framework, not for general game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| HideEventTitle(float) | constructor | O(1) | Creates a new packet for outbound transmission. |
| getId() | int | O(1) | Returns the static network ID (215) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes the packet's state into a byte buffer. For internal use by the network encoder. |
| computeSize() | int | O(1) | Returns the fixed size of the packet in bytes (4). |
| deserialize(ByteBuf, int) | static HideEventTitle | O(1) | Creates a packet instance by reading from a byte buffer. For internal use by the network decoder. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(1) | Verifies if a buffer contains enough data to form a valid packet. |

## Integration Patterns

### Standard Usage
This packet is typically handled by a dedicated listener or handler on the client side. The handler extracts the fade duration and forwards it to the UI management system.

```java
// Example of a client-side packet handler
public void handleHideEventTitle(HideEventTitle packet) {
    // Retrieve the UI manager from the game context
    UIManager uiManager = game.getUIManager();

    // Instruct the UI manager to perform the action
    uiManager.hideEventTitle(packet.fadeOutDuration);
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store references to HideEventTitle packets. They represent a point-in-time event. Extract the data and discard the packet object.
- **Manual Serialization/Deserialization:** Never call *serialize* or *deserialize* directly in game logic. These methods are exclusively for the network pipeline encoders and decoders. The network layer is responsible for managing the byte-level representation.
- **Reusing Instances:** Do not modify and re-send the same packet instance. Always create a new instance for each distinct command to avoid unintended side effects.

## Data Pipeline
The flow of this data is unidirectional from the server's game logic to the client's UI system.

> **Outbound Flow (Server):**
> Game Event Trigger -> `new HideEventTitle(1.0f)` -> Client Connection's Send Queue -> Network Encoder -> **HideEventTitle.serialize()** -> TCP/IP Stack

> **Inbound Flow (Client):**
> TCP/IP Stack -> Netty ByteBuf -> Packet Decoder -> **HideEventTitle.deserialize()** -> Packet Handler -> UIManager -> Render Thread Update

