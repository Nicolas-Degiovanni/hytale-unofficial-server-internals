---
description: Architectural reference for CustomPage
---

# CustomPage

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class CustomPage implements Packet {
```

## Architecture & Concepts
The CustomPage class is a network packet structure that defines a complete, server-driven user interface screen. It acts as a blueprint, containing all the necessary commands and event bindings for a client to construct, display, and interact with a custom UI page without requiring a client-side code update.

This packet is a cornerstone of Hytale's flexible UI system, allowing servers to dynamically send new interfaces to players for features like shops, information panels, or custom game mechanics.

The binary layout of this packet is highly optimized for performance and security. It employs a fixed-size header block that contains a nullability bitfield and integer offsets pointing to the location of variable-length data (like strings and arrays) within the packet body. This design allows for:
1.  **Efficient Validation:** The `validateStructure` method can check offsets and lengths without deserializing the entire payload, quickly rejecting malformed or malicious packets.
2.  **Non-Sequential Access:** Decoders can read specific fields without needing to parse all preceding data.
3.  **Predictable Header Size:** The initial block is always a known size (16 bytes), simplifying the initial stages of packet processing.

## Lifecycle & Ownership
- **Creation:** An instance of CustomPage is created on the server when game logic dictates that a client's UI must be updated or replaced. On the client, it is created exclusively by the network layer's packet deserialization pipeline via the static `deserialize` factory method.
- **Scope:** This object is ephemeral and has an extremely short lifespan. On the server, it exists only for the duration of the serialization and network dispatch process. On the client, it exists only until the UI system has consumed its data to build the interface.
- **Destruction:** The object is eligible for garbage collection immediately after being processed by its respective system (network serializer on server, UI controller on client). There is no persistent state held within the object across ticks or network events.

## Internal State & Concurrency
- **State:** The CustomPage object is a mutable data container. Its public fields are intended to be populated once upon creation and then read during a single processing operation.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, such as a server's main logic thread or a client's network event loop. Concurrent modification from multiple threads will result in unpredictable behavior and data corruption. All synchronization must be handled externally.

## API Surface
The primary contract of this class is its static serialization and validation interface, not its instance members.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | CustomPage | O(N) | **[Factory]** Constructs a CustomPage instance by reading from a raw ByteBuf. Throws ProtocolException on data corruption. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a security and integrity check on the raw buffer data before attempting full deserialization. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. |
| computeBytesConsumed(buf, offset) | int | O(N) | Calculates the total size of a packet within a buffer without performing a full deserialization. |

## Integration Patterns

### Standard Usage
The CustomPage packet is handled entirely by the server's network layer and the client's UI system. A developer typically creates and sends it, but does not interact with the deserialization process.

```java
// Server-side: Creating and sending a new UI page
CustomUICommand[] commands = { ... };
CustomUIEventBinding[] bindings = { ... };

CustomPage welcomeScreen = new CustomPage(
    "player_welcome",
    true,
    true,
    CustomPageLifetime.CantClose,
    commands,
    bindings
);

// The network manager serializes and sends the packet to the client
player.getConnection().sendPacket(welcomeScreen);
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Do not modify a CustomPage instance after it has been sent or processed. These objects are intended to be immutable after creation. Reusing instances can lead to subtle bugs where old data is unintentionally resent.
- **Ignoring Validation:** Never deserialize a CustomPage packet from an untrusted source without first calling `validateStructure`. Bypassing this step exposes the client or server to buffer overflow attacks and denial-of-service vulnerabilities from maliciously crafted packets.
- **Manual Serialization:** Do not attempt to write the fields to a buffer manually. The binary format is complex, involving null-bitmasks and data offsets. Always use the provided `serialize` method to ensure correctness.

## Data Pipeline
The CustomPage object is a data payload that flows from the server's game logic to the client's rendering engine.

> Flow:
> Server Game Logic -> **new CustomPage(...)** -> Network Serialization -> TCP/IP Stack -> Client Network Deserialization -> **CustomPage.deserialize(...)** -> Client UI System -> UI Render Tree Update

