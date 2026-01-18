---
description: Architectural reference for BuilderToolGeneralAction
---

# BuilderToolGeneralAction

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Data Transfer Object (Packet)

## Definition
```java
// Signature
public class BuilderToolGeneralAction implements Packet {
```

## Architecture & Concepts
The BuilderToolGeneralAction class is a lightweight, special-purpose Data Transfer Object, commonly referred to as a Packet within the Hytale network protocol. Its sole function is to encapsulate and transport a single, simple command related to the in-game builder tools between the client and the server.

This class is a fundamental component of the network protocol layer. It does not contain any game logic. Instead, it serves as a structured message format that the game client and server have agreed upon. The static fields within the class, such as PACKET_ID and FIXED_BLOCK_SIZE, are critical metadata. This metadata is consumed by the protocol's higher-level codec and dispatcher systems to correctly identify, parse, and route incoming byte streams to the appropriate handler.

Due to its fixed size of one byte, this packet is optimized for high-frequency, low-latency actions like selecting a tool mode or confirming a selection, where performance is paramount.

## Lifecycle & Ownership
- **Creation:** An instance is created under two primary circumstances:
    1.  **On the sending side (e.g., Client):** Instantiated by a higher-level system, such as an input manager or UI controller, in response to a direct player action. The desired BuilderToolAction is passed into the constructor.
    2.  **On the receiving side (e.g., Server):** Instantiated by the protocol's deserialization engine. The static `deserialize` method is invoked by a network pipeline handler when a byte stream with packet ID 412 is identified.

- **Scope:** The object's lifetime is extremely brief and transient. It is designed to exist only for the duration of a single network event processing cycle. Once its internal `action` data has been extracted and passed to the relevant game logic handler, the BuilderToolGeneralAction object is no longer referenced and becomes eligible for garbage collection.

- **Destruction:** Managed automatically by the Java Garbage Collector. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency
- **State:** The class holds a single piece of mutable state: the `action` field. While technically mutable, instances of this packet should be treated as immutable after their initial creation. Modifying a packet after it has been queued for sending or after it has been deserialized is a severe anti-pattern.

- **Thread Safety:** **This class is not thread-safe.** Packet objects are designed to be created, processed, and discarded within the confines of a single thread, typically a Netty I/O worker thread or the main game logic thread. Sharing instances of this packet across threads without explicit, external synchronization will result in undefined behavior and is strictly forbidden.

## API Surface
The public contract is focused on serialization, deserialization, and protocol identification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static packet identifier (412) used for protocol dispatch. |
| serialize(ByteBuf) | void | O(1) | Writes the single-byte action value into the provided Netty ByteBuf. |
| deserialize(ByteBuf, int) | BuilderToolGeneralAction | O(1) | **Static Factory.** Reads one byte from the buffer and constructs a new packet instance. |
| computeSize() | int | O(1) | Returns the fixed network size of the packet, which is always 1 byte. |

## Integration Patterns

### Standard Usage
The class is used to signal a simple, parameter-less action from the client to the server. The sending system constructs the packet, and the receiving system's handler consumes it.

```java
// Client-side: Sending a "set position 1" action
BuilderToolAction action = BuilderToolAction.SelectionPosition1;
BuilderToolGeneralAction packet = new BuilderToolGeneralAction(action);

// The network manager handles the actual serialization and transmission
networkManager.sendPacket(packet);

// Server-side: Inside a packet handler after deserialization
public void handle(BuilderToolGeneralAction packet) {
    Player player = getPlayerForConnection();
    BuilderToolSystem system = player.getBuilderToolSystem();
    system.processAction(packet.action);
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not cache and reuse packet objects. They are inexpensive to create and are not designed for reuse. Reusing a packet can lead to state corruption if it is mutated while another part of the system holds a reference to it.

- **Manual Serialization/Deserialization:** Do not manually read from or write to a ByteBuf to simulate this packet. Always use the provided `serialize` and static `deserialize` methods to ensure correctness and forward compatibility with the protocol.

- **State Modification After Creation:** Do not modify the `action` field after the packet has been constructed and passed to the network layer. Treat the object as immutable once created.

## Data Pipeline
The flow of data encapsulated by this packet is linear and unidirectional for a single transaction.

> Flow:
> Player Input (Client) -> UI/Input Controller -> **new BuilderToolGeneralAction()** -> NetworkManager -> Protocol Codec (serialize) -> TCP/IP Stack -> Server -> Protocol Codec (deserialize) -> **BuilderToolGeneralAction instance** -> Packet Dispatcher -> Game Logic Handler

