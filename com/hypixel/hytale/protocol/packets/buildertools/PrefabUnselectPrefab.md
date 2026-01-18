---
description: Architectural reference for PrefabUnselectPrefab
---

# PrefabUnselectPrefab

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Transient

## Definition
```java
// Signature
public class PrefabUnselectPrefab implements Packet {
```

## Architecture & Concepts
The PrefabUnselectPrefab class represents a network packet that functions as a pure **signal**. It is a command object within the builder tools protocol, designed to communicate a single, unambiguous event: the user has deselected the currently active prefab in the editor.

Architecturally, this packet is an example of a zero-payload message. Its entire meaning is conveyed by its type, identified by its unique PACKET_ID (417). The network protocol layer uses this ID to route the signal to the appropriate handler. By having no data fields, it is extremely efficient, consuming minimal network bandwidth and requiring negligible CPU for serialization or deserialization.

This class is a fundamental component of the client-server communication model for real-time creative tools, enabling state synchronization based on user actions. Its existence implies that the server should update its model of the player's builder tool state to reflect that no prefab is currently selected for placement.

### Lifecycle & Ownership
- **Creation:** An instance is created on the client-side by the UI or input handling logic for the builder tools. This typically occurs in response to a direct user action, such as pressing the escape key or clicking a "deselect" button. It is never managed by a dependency injection container.

- **Scope:** The object's lifetime is exceptionally brief and ephemeral. It exists only for the immediate task of being passed to the network serialization pipeline. On the receiving end, a new instance is created during deserialization, processed by a handler, and then immediately becomes eligible for garbage collection.

- **Destruction:** There is no manual destruction. The Java Garbage Collector reclaims the memory for the object shortly after it has been serialized (on the sender) or processed (on the receiver).

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class contains no instance fields. All instances of PrefabUnselectPrefab are functionally identical and interchangeable. The static final fields define the packet's metadata, which is shared across all instances.

- **Thread Safety:** **Inherently Thread-Safe**. As a stateless, immutable object, it can be passed between threads without any risk of data corruption or race conditions. In practice, packet creation and processing are typically confined to a single network or game logic thread to maintain protocol order.

## API Surface
The public API is dictated by the Packet interface contract and is used exclusively by the network protocol engine.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network ID (417) for this packet type. |
| serialize(ByteBuf) | void | O(1) | Writes nothing to the buffer. This packet has no payload. |
| deserialize(ByteBuf, int) | PrefabUnselectPrefab | O(1) | Static factory. Creates a new instance without reading from the buffer. |
| computeSize() | int | O(1) | Returns 0, indicating a zero-byte payload. |

## Integration Patterns

### Standard Usage
This packet should be instantiated and sent via a network service when the user deselects a prefab. It is a fire-and-forget operation.

```java
// Executed within a client-side input handler or UI event listener
// when the user deselects a prefab.
PacketSender netService = context.getService(PacketSender.class);

// Create and immediately dispatch the signal packet.
netService.send(new PrefabUnselectPrefab());
```

### Anti-Patterns (Do NOT do this)
- **Payload Extension:** Do not attempt to subclass or modify this packet to include data. Its core design principle is its lack of a payload. If you need to send data related to prefab selection, a new and distinct packet type must be created.

- **Stateful Caching:** Do not cache and reuse instances of this packet. While technically possible due to its stateless nature, the standard protocol pattern is to create a new packet for each discrete event. This avoids potential side effects in more complex systems and keeps the logic clear.

- **Expecting Data on Receive:** The handler for this packet must not attempt to read any data from the network buffer. The protocol engine will advance the buffer's read index by zero bytes, as correctly reported by `computeBytesConsumed`.

## Data Pipeline
The flow of this packet is a simple, one-way signal from the client to the server to notify it of a state change in the user's toolset.

> Flow:
> Client Input Event (e.g., Key Press) -> Builder Tool Logic -> **new PrefabUnselectPrefab()** -> Network Serialization Engine -> TCP/IP Stack -> Server Network Deserialization -> Packet Dispatcher -> Server-Side Builder Tool State Handler

