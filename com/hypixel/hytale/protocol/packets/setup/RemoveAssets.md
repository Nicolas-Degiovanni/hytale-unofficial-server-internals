---
description: Architectural reference for the RemoveAssets network packet.
---

# RemoveAssets

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class RemoveAssets implements Packet {
```

## Architecture & Concepts

The RemoveAssets class is a network **Packet** that functions as a Data Transfer Object (DTO) within the Hytale protocol. Its sole purpose is to encapsulate a command from the server to the client, instructing it to unload a specific list of game assets from memory.

This packet is a critical component of the client-server **setup** and asset synchronization lifecycle. It allows the server to dynamically manage the client's memory footprint by purging assets that are no longer required for the current game state, such as those from a previous zone or minigame. It operates purely at the network protocol layer and has no inherent game logic. Its contents are deserialized and immediately passed to a higher-level system, typically an AssetManager, for execution.

## Lifecycle & Ownership

-   **Creation:**
    -   On the **server**, an instance is created programmatically when the server logic determines a client must unload assets. It is populated with the target Asset identifiers and serialized into the network pipeline.
    -   On the **client**, an instance is created by a network deserializer (e.g., a Netty ChannelInboundHandler) when a raw byte buffer with Packet ID 27 is received.

-   **Scope:** Extremely short-lived and transactional. The object exists only for the brief moment between deserialization and processing by a packet handler. It is not retained or cached.

-   **Destruction:** The object is eligible for garbage collection immediately after the corresponding packet handler has finished processing it. There are no manual resource management or cleanup steps required.

## Internal State & Concurrency

-   **State:** The class holds a mutable array of Asset objects. While technically mutable, it should be treated as an immutable message after its initial creation (on the server) or deserialization (on the client). Modifying its state after it has entered a processing pipeline can lead to undefined behavior.

-   **Thread Safety:** **This class is not thread-safe.** Packet processing is expected to occur on a single network thread, such as a Netty event loop. Any handoff to other threads (e.g., the main game loop or an asset loading thread) must be done through a thread-safe queue or dispatcher. Direct concurrent access will result in data corruption.

## API Surface

The public API is dominated by static methods for protocol-level serialization and validation, reflecting its role as a DTO.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the static network identifier for this packet, which is 27. |
| serialize(ByteBuf) | void | O(N) | Encodes the internal asset array into a binary format in the provided ByteBuf. Throws ProtocolException on constraint violations. |
| deserialize(ByteBuf, int) | static RemoveAssets | O(N) | Decodes a RemoveAssets packet from a ByteBuf. Throws ProtocolException if the data is malformed or violates constraints. |
| computeSize() | int | O(N) | Calculates the exact number of bytes this packet will occupy when serialized. Used for buffer pre-allocation. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a lightweight check on a buffer to confirm if it contains a structurally valid RemoveAssets packet without full deserialization. |

*N = number of assets in the array*

## Integration Patterns

### Standard Usage

The packet is intended to be handled by a dedicated network listener which dispatches the asset removal command to the appropriate service.

**Server-Side (Sending)**
```java
// Example: Sending a command to remove two assets
Asset asset1 = new Asset(...);
Asset asset2 = new Asset(...);

RemoveAssets packet = new RemoveAssets(new Asset[]{asset1, asset2});
playerConnection.sendPacket(packet);
```

**Client-Side (Receiving)**
```java
// Example: In a network handler class
public void handle(RemoveAssets packet) {
    if (packet.asset != null) {
        AssetManager assetManager = context.getService(AssetManager.class);
        assetManager.unloadAssets(packet.asset);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **State Reuse:** Do not hold a reference to a RemoveAssets packet after it has been handled. These objects are transient and should not be cached or reused.
-   **Cross-Thread Modification:** Do not deserialize a packet on a network thread and then modify its contents on a different thread. Pass the immutable data, not the packet object itself.
-   **Manual Serialization:** Avoid calling serialize directly. This is the responsibility of the network pipeline's encoder. Manually managing byte buffers is error-prone.

## Data Pipeline

The flow of data encapsulated by this packet is unidirectional from server to client.

> **Server Flow:**
> Game Logic -> **RemoveAssets (new)** -> Packet Serializer -> Netty Channel -> Network
>
> **Client Flow:**
> Network -> Netty Channel -> Packet Deserializer -> **RemoveAssets (instance)** -> Packet Handler -> AssetManager

