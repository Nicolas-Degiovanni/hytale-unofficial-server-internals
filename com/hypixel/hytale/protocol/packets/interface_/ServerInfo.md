---
description: Architectural reference for ServerInfo
---

# ServerInfo

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class ServerInfo implements Packet {
```

## Architecture & Concepts

The ServerInfo class is a concrete implementation of the Packet interface, serving as a pure Data Transfer Object. It represents a single, well-defined network message, identified by the static PACKET ID 223. Its primary responsibility is to model the in-memory representation of server metadata—specifically its name, message of the day (MOTD), and maximum player capacity.

This class is fundamental to the client-server connection lifecycle, used during the initial server list ping and handshake sequences. It provides the client with the necessary information to display a server in its server browser UI before a full connection is established.

The binary serialization format is custom and highly optimized for network performance. It employs a structure composed of a fixed-size block and a variable-size block:
*   **Fixed Block:** Contains a bitmask for nullable fields (nullBits) and fixed-size data like maxPlayers. It also contains integer offsets that point to the location of variable-sized data.
*   **Variable Block:** Appended after the fixed block, this section contains the actual byte data for variable-length fields like serverName and motd.

This design allows for extremely fast, non-sequential parsing. A parser can read the fixed block to determine which fields are present and where they are located in the buffer without needing to read all preceding data.

## Lifecycle & Ownership

-   **Creation:**
    -   **Receiving (Client):** Instantiated exclusively by the network protocol layer. When an incoming data stream is decoded and identified as Packet 223, the static `deserialize` factory method is invoked to construct a ServerInfo object from the raw ByteBuf.
    -   **Sending (Server):** Instantiated directly via `new ServerInfo(...)` by server-side logic. The object is populated with the current server state and then passed to the network pipeline for serialization.

-   **Scope:** Transient and extremely short-lived. An instance of ServerInfo exists only for the brief moment it takes to process a single network message. It is created, read from (or written to), and immediately becomes eligible for garbage collection.

-   **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods. Once all references are dropped, typically after a packet handler completes its execution, the object is destroyed.

## Internal State & Concurrency

-   **State:** Mutable. The public fields `serverName`, `motd`, and `maxPlayers` can be modified after the object has been constructed. It is a simple data container with no internal logic for state management.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty I/O worker thread. It contains no internal locks or synchronization mechanisms. Accessing or modifying a ServerInfo instance from multiple threads concurrently will result in undefined behavior and data corruption. All synchronization must be handled externally.

## API Surface

The public API is dominated by static methods for serialization and validation, reflecting its role as a data-centric packet definition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static ServerInfo | O(N) | **Primary Factory.** Constructs a ServerInfo object by parsing a binary representation from a ByteBuf. N is the total length of string data. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Encodes the object's state into the provided ByteBuf according to the custom binary protocol. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the current state of the object. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a low-cost structural validation on a buffer without full deserialization. Crucial for network security to reject malformed packets early. |

## Integration Patterns

### Standard Usage

**Receiving Data (Client-Side)**
The object is typically created by a central packet decoder and passed to an event bus or a specific handler for processing.

```java
// Executed within a Netty channel handler or network dispatcher
// The packet ID has already been identified as 223.
ServerInfo info = ServerInfo.deserialize(incomingBuffer, bufferOffset);

// Post the immutable data to the rest of the application
// to be processed by the UI thread.
UiEventManager.post(new ServerListEntryUpdate(info.serverName, info.motd, info.maxPlayers));
```

**Sending Data (Server-Side)**
An instance is created, populated, and passed to the network channel, where a pipeline encoder will automatically call the `serialize` method.

```java
// In a server-side handler responding to a client ping
ServerInfo response = new ServerInfo("Hytale World One", "Welcome, adventurer!", 100);

// The Netty pipeline is configured with an encoder that will find the
// appropriate serializer for the ServerInfo object.
channel.writeAndFlush(response);
```

### Anti-Patterns (Do NOT do this)

-   **Object Reuse:** Do not modify and resend the same ServerInfo instance. This is not safe, especially in an asynchronous environment. Always create a new instance for each discrete message to ensure message integrity.
-   **Cross-Thread Access:** Do not create a ServerInfo object on a game logic thread and pass its reference to a network thread for serialization without a deep copy or proper synchronization. The mutable nature of the object makes it highly susceptible to race conditions.
-   **Manual Serialization:** Avoid calling `serialize` directly. The engine's network layer abstracts this via pipeline encoders. Direct calls can lead to buffer management errors and bypass framework-level optimizations.

## Data Pipeline

The ServerInfo object is a data record that flows through the network stack.

**Inbound (Client-Side Perspective)**
> Flow:
> Raw TCP Bytes → Netty ByteBuf → Protocol Framer & Decoder → **ServerInfo.deserialize** → ServerInfo Object → Packet Handler → UI Thread (Update Server List)

**Outbound (Server-Side Perspective)**
> Flow:
> Server Ping Logic → `new ServerInfo(...)` → **ServerInfo Object** → Network Channel → Packet Encoder (calls `serialize`) → Netty ByteBuf → Raw TCP Bytes

