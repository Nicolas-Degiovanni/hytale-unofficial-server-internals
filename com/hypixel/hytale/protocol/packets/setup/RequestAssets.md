---
description: Architectural reference for RequestAssets
---

# RequestAssets

**Package:** com.hypixel.hytale.protocol.packets.setup
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class RequestAssets implements Packet {
```

## Architecture & Concepts
The RequestAssets class is a message contract within the client-server setup protocol. It does not represent a service or a long-lived component; its sole function is to encapsulate a client's request for a specific list of game assets from a server. This class is a critical element of the asset synchronization pipeline that occurs during the client connection sequence.

As a Packet implementation, it defines the precise binary serialization format for an asset request. The design prioritizes network efficiency and processing speed, utilizing a custom binary layout that includes a nullability bitfield and variable-length integer encoding (VarInt) for the asset array count. This minimizes payload size and reduces deserialization overhead.

Architecturally, RequestAssets acts as an immutable data carrier that crosses the boundary between the high-level Asset Management system and the low-level Network Transport layer.

## Lifecycle & Ownership
The lifecycle of a RequestAssets object is exceptionally brief and tied directly to a single network operation.

-   **Creation:**
    -   **Client (Outbound):** Instantiated by a client-side asset synchronization service. The service determines which assets are missing or outdated, populates a new RequestAssets instance with the required Asset identifiers, and passes it to the network layer for serialization.
    -   **Server (Inbound):** Instantiated by the server's network protocol decoder. When a packet with ID 23 is received, the decoder invokes the static `deserialize` factory method, which reads from the network ByteBuf and constructs the RequestAssets object.

-   **Scope:** Transient. An outbound instance exists only for the duration of the serialization and network write call. An inbound instance exists only until the corresponding network handler has finished processing the request. It is not designed to be cached or referenced outside the scope of its immediate processing logic.

-   **Destruction:** The object becomes eligible for garbage collection immediately after use. On the client, this is after serialization. On the server, this is after the asset request has been dispatched to the asset delivery system.

## Internal State & Concurrency
-   **State:** The internal state consists of a single mutable field: the `assets` array. The object itself does not perform any caching, lazy initialization, or state transformation. It is a plain data holder.

-   **Thread Safety:** This class is **not thread-safe**. The public `assets` field is mutable and accessed without any synchronization primitives. It is designed to be created, populated, and processed by a single thread, typically a Netty I/O worker thread.

    **Warning:** Sharing a RequestAssets instance across threads without external locking will result in race conditions and data corruption, particularly if one thread modifies the `assets` array while another is serializing it.

## API Surface
The public API is divided between instance methods for outbound packets and static methods for inbound data processing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| serialize(ByteBuf) | void | O(N) | Serializes the object's state into the provided ByteBuf. Throws ProtocolException if the asset array is too large. |
| computeSize() | int | O(N) | Calculates the final size in bytes of the serialized packet without performing the write. |
| deserialize(ByteBuf, int) | static RequestAssets | O(N) | Constructs a RequestAssets object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check of the buffer to validate if the data represents a well-formed packet. Does not instantiate the object. |

*N = number of assets in the `assets` array.*

## Integration Patterns

### Standard Usage
The class is intended to be used within the network pipeline. On the server, a channel handler receives the fully formed object after it has been decoded.

```java
// Example of a server-side Netty handler
public class SetupPacketHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof RequestAssets) {
            RequestAssets request = (RequestAssets) msg;
            if (request.assets != null) {
                // Dispatch the request to the asset streaming service
                // to begin sending asset data back to the client.
                AssetStreamingService.queueAssetsForClient(ctx.channel(), request.assets);
            }
        } else {
            ctx.fireChannelRead(msg);
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Do not modify and resend a RequestAssets instance. For each new request, create a new instance to ensure message integrity.
-   **Long-Term Storage:** Do not hold references to RequestAssets objects in caches or as part of game state. They represent a point-in-time request and should be discarded after processing.
-   **Cross-Thread Access:** Avoid creating the packet on a main game thread and passing it to a network thread for sending. This pattern is fragile. The safest approach is to pass the raw data (the Asset array) to the network thread and construct the RequestAssets object there, just before serialization.

## Data Pipeline
The flow of data through this component is linear and unidirectional for a given context (client or server).

> **Client Outbound Flow:**
> Asset Synchronization Logic → `Asset[]` → **new RequestAssets(assets)** → PacketEncoder.serialize() → Netty Channel Pipeline → TCP Socket

> **Server Inbound Flow:**
> TCP Socket → Netty Channel Pipeline → PacketDecoder.deserialize() → **RequestAssets instance** → SetupPacketHandler → Asset Streaming Service

