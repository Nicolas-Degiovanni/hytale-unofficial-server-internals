---
description: Architectural reference for ClientReferral
---

# ClientReferral

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Transient Data Model

## Definition
```java
// Signature
public class ClientReferral implements Packet {
```

## Architecture & Concepts
The ClientReferral class is a data-only structure that represents a specific network message within Hytale's authentication and server-joining protocol. It is not a service or manager; it is a Plain Old Java Object (POJO) whose sole responsibility is to model the data for packet ID 18.

Architecturally, this packet serves as a critical link in the server-to-server handover process. A typical scenario involves a central authentication or lobby server verifying a player's credentials and then "referring" them to a specific game world server. This class encapsulates the necessary information for that transition: the target server's address (**hostTo**) and an opaque data payload (**data**), which likely contains a single-use token or session ticket for the game server to validate.

The on-wire format is highly optimized for network performance. It employs a custom serialization scheme that includes:
1.  A **null-bitfield** at the start of the payload to indicate which of the nullable fields are present, saving bandwidth by not sending empty data.
2.  An **offset table** for variable-length fields. Instead of writing fields sequentially, the packet contains fixed-size integer offsets pointing to the location of each variable-sized field within the packet's data block. This allows for efficient parsing and validation.

## Lifecycle & Ownership
-   **Creation:** An instance of ClientReferral is created under two primary conditions:
    1.  **Inbound:** The protocol layer's deserialization pipeline instantiates it via the static **deserialize** factory method when an incoming network buffer containing packet ID 18 is processed.
    2.  **Outbound:** Server-side application logic (e.g., a login service) instantiates it directly using its constructor to prepare a referral message to be sent to a client.

-   **Scope:** The object's lifetime is exceptionally short and transient. It exists only for the duration of its processing within a single network event handler or until it has been serialized into an outbound network buffer.

-   **Destruction:** The object is managed by the Java Garbage Collector. As it is a simple data holder with no external resources, it is eligible for collection as soon as all references to it are dropped, which typically occurs upon exiting the network handler method.

## Internal State & Concurrency
-   **State:** The class is a mutable data container. Its public fields, **hostTo** and **data**, can be modified after instantiation. It does not cache any information and its state is a direct representation of the data it holds.

-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. It is designed to be created, populated, and processed within the confines of a single thread, such as a Netty I/O worker thread. Sharing instances across multiple threads without external synchronization will result in undefined behavior and data corruption.

## API Surface
The public contract is focused on serialization, validation, and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | ClientReferral | O(N) | **Static Factory.** Constructs a ClientReferral instance by reading from a ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the protocol's custom format. |
| validateStructure(buf, offset) | ValidationResult | O(1) | **Static Validator.** Performs a fast, non-allocating check on a buffer to verify if it contains a structurally valid packet. |
| computeSize() | int | O(1) | Calculates the exact number of bytes required to serialize the current state of the object. |
| computeBytesConsumed(buf, offset) | int | O(1) | **Static Utility.** Calculates the total length of a serialized packet within a buffer without full deserialization. |

*Complexity O(N) refers to the length of the byte array in the **data** field.*

## Integration Patterns

### Standard Usage
The primary integration point is with the network layer. Application logic creates and populates the packet, then passes it to a network channel for transmission.

```java
// Example: A login server preparing to refer a client
HostAddress gameServerAddress = new HostAddress("192.168.1.100", 25565);
byte[] sessionTicket = generateSecureSessionTicket();

ClientReferral referralPacket = new ClientReferral(gameServerAddress, sessionTicket);

// The network layer (e.g., Netty Channel) would then handle the serialization
playerConnection.sendPacket(referralPacket);
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not hold onto and reuse ClientReferral instances for multiple messages. They are lightweight objects, and re-using them can lead to subtle bugs where old data is accidentally sent. Always create a new instance for each new message.
-   **Manual Serialization:** Never attempt to read or write the packet's data to a buffer manually. The serialization format is complex, involving null bitfields and data offsets. Always use the provided **serialize** and **deserialize** methods to ensure protocol compliance.
-   **Cross-Thread Modification:** Do not create a ClientReferral object on one thread and modify it on another without proper synchronization. This will lead to race conditions. The object should be confined to the thread that sends or receives it.

## Data Pipeline
The ClientReferral object is a data payload that moves through the network stack.

**Outbound Flow (Server Sending to Client):**
> Game Logic -> `new ClientReferral()` -> Network Channel -> **ClientReferral.serialize()** -> ByteBuf -> TCP Socket

**Inbound Flow (Client Receiving from Server):**
> TCP Socket -> ByteBuf -> Protocol Decoder -> **ClientReferral.deserialize()** -> ClientReferral Instance -> Network Event Handler

