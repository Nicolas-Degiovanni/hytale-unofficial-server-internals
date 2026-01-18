---
description: Architectural reference for AssetEditorAuthorization
---

# AssetEditorAuthorization

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorAuthorization implements Packet {
```

## Architecture & Concepts
The AssetEditorAuthorization packet is a simple, low-level Data Transfer Object (DTO) used within the Hytale network protocol. Its sole purpose is to communicate a binary authorization state from the server to the client, indicating whether the connected user is permitted to use the in-game asset editor.

This class is not a service or manager; it is a piece of transient data. It represents a single, atomic message in the asset editor's control flow. The protocol's serialization and deserialization engine uses the static constants defined in this class (such as PACKET_ID and FIXED_BLOCK_SIZE) to efficiently identify and parse the packet from the raw network stream. It acts as a server-authoritative command that directly influences client-side UI and feature availability.

## Lifecycle & Ownership
- **Creation:** An instance is created under two distinct circumstances:
    1. **Sending:** The server-side logic (e.g., a permission-checking service after login) instantiates this packet to be sent to a client.
    2. **Receiving:** The client-side network layer's packet deserializer instantiates this packet after reading its ID and data from an incoming Netty ByteBuf.
- **Scope:** The object's lifetime is exceptionally short and confined to the scope of a single network event processing cycle. It is created, processed by a handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** There is no explicit destruction. The object is released once the packet handler that receives it completes its execution. **WARNING:** Holding a reference to this packet beyond the handler's scope is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The class holds a single mutable boolean field, *canUse*. Its state is a direct representation of the data read from or written to the network buffer. It performs no caching and has no side effects.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created and processed within a single thread, typically a Netty I/O worker thread or the main game thread. Do not share instances of this packet across threads; its mutable state can lead to race conditions.

## API Surface
The public contract is primarily for the network protocol framework, not for general application logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AssetEditorAuthorization(boolean) | constructor | O(1) | Creates a packet for serialization and transmission. |
| canUse | boolean | O(1) | Public field representing the authorization status. |
| serialize(ByteBuf) | void | O(1) | Writes the packet's state into a network buffer. |
| deserialize(ByteBuf, int) | static AssetEditorAuthorization | O(1) | Constructs a packet from a network buffer. |

## Integration Patterns

### Standard Usage
This packet is intended to be processed by a dedicated packet handler. The handler extracts the boolean value and updates a more permanent, thread-safe state manager.

```java
// Example of a client-side packet handler
public class AssetEditorAuthorizationHandler implements PacketHandler<AssetEditorAuthorization> {
    private final ClientState clientState;

    // ... constructor ...

    @Override
    public void handle(AssetEditorAuthorization packet) {
        // The packet's data is immediately transferred to a proper state manager.
        // The packet object itself is then discarded.
        this.clientState.setAssetEditorEnabled(packet.canUse);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Usage:** Do not store an instance of AssetEditorAuthorization as a member variable in a service. It is an event, not a state container. Copy its value out immediately.
- **Client-Side Instantiation:** Client code should never create this packet using `new AssetEditorAuthorization()`. It is a server-to-client message. Doing so has no effect and indicates a misunderstanding of the protocol.
- **Cross-Thread Sharing:** Do not pass this object from the network thread to another thread. If the data is needed elsewhere, pass the primitive `canUse` boolean value instead.

## Data Pipeline
The data flow for this packet is unidirectional from server to client.

> Flow:
> Server Permission Check -> **AssetEditorAuthorization(true/false)** -> Protocol Serializer -> TCP/IP Stack -> Client Network Layer -> Protocol Deserializer -> **AssetEditorAuthorization** instance -> Packet Handler -> Client State Update -> UI Enable/Disable

