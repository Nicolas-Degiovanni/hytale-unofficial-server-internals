---
description: Architectural reference for SetServerAccess
---

# SetServerAccess

**Package:** com.hypixel.hytale.protocol.packets.serveraccess
**Type:** Transient

## Definition
```java
// Signature
public class SetServerAccess implements Packet {
```

## Architecture & Concepts
The **SetServerAccess** class is a Data Transfer Object (DTO) that represents a single, specific message within the Hytale network protocol. Its sole purpose is to define the structure and provide the serialization/deserialization logic for a command that configures a game server's public accessibility.

This class is a fundamental component of the protocol layer, acting as a strict data contract between the client and server. It does not contain any game logic, state management, or business rules. Instead, it serves as a low-level data container that is created, populated, and passed to the network pipeline for transmission, or created by the network pipeline upon receiving data. Its design is optimized for performance and precise byte-level manipulation, leveraging Netty's **ByteBuf** for direct memory operations.

Key architectural characteristics include:
- **Immutability by Convention:** While its fields are technically mutable, it is designed to be treated as an immutable value object after its initial creation.
- **Static Factories:** The primary ingress path for network data is the static **deserialize** method, which acts as a factory for creating instances from a raw byte buffer.
- **Self-Contained Logic:** All logic for validation, size calculation, serialization, and deserialization is encapsulated within the class itself, preventing protocol-specific code from leaking into other layers of the application.

## Lifecycle & Ownership
- **Creation:** An instance of **SetServerAccess** is created under two distinct circumstances:
    1. **Inbound:** The network layer's packet decoder invokes the static **deserialize** method when it identifies an incoming packet with ID 252. This creates a new object from the network stream.
    2. **Outbound:** Application logic (e.g., a server command handler or a client-side UI) instantiates it directly using `new SetServerAccess(...)` to prepare a message for sending.
- **Scope:** The object's lifetime is extremely short and transactional. It exists only for the duration of a single operation: either being encoded into a **ByteBuf** for sending or being decoded and processed by a packet handler.
- **Destruction:** It is a lightweight object with no external resources. Once processing is complete, it is dereferenced and becomes eligible for standard garbage collection. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state is minimal and fully mutable, consisting of an **Access** enum and a nullable **String** for the password. It does not cache any data or maintain any state beyond its own fields.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created, populated, and processed within the confines of a single thread, typically a Netty I/O worker thread or a main game logic thread. Concurrent modification will result in race conditions and unpredictable serialization output.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | SetServerAccess | O(N) | **[Static Factory]** Constructs an object from a byte buffer. N is the password length. Throws **ProtocolException** on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided byte buffer. N is the password length. |
| computeSize() | int | O(N) | Calculates the exact number of bytes required to serialize the object. N is the password length. |
| validateStructure(buffer, offset) | ValidationResult | O(N) | **[Static]** Performs a read-only check on a buffer to ensure it contains a valid object structure without full deserialization. N is the password length. |
| getId() | int | O(1) | Returns the constant network ID (252) for this packet type. |

## Integration Patterns

### Standard Usage
**SetServerAccess** is intended for direct instantiation when sending data. It is then passed to a network-aware service or channel for serialization and transmission.

```java
// Example: A server-side handler setting the server to be private with a password.
import com.hypixel.hytale.network.NetworkManager; // Fictional network manager

Access newAccess = Access.Private;
String newPassword = "a-secure-password";

// 1. Create the packet DTO
SetServerAccess packet = new SetServerAccess(newAccess, newPassword);

// 2. Pass to the network layer for dispatch
networkManager.sendPacketToClient(clientId, packet);
```

### Anti-Patterns (Do NOT do this)
- **Object Re-use:** Do not attempt to pool or re-use **SetServerAccess** instances. They are cheap to allocate and their internal state is not designed for reset. Always create a new instance for each message.
- **Modification After Send:** Modifying a **SetServerAccess** object after it has been passed to the network layer for sending is a severe anti-pattern. The network pipeline may read its state at any time, leading to race conditions. Treat it as immutable after creation.
- **Cross-Thread Sharing:** Never pass an instance of this object between threads. If data needs to be transferred, extract the primitive values (access level, password) and construct a new packet on the target thread.

## Data Pipeline
The class serves as a data model at a specific point in the network data flow.

> **Outbound Flow (Sending):**
> Application Logic -> `new SetServerAccess()` -> **SetServerAccess.serialize()** -> Netty Channel Pipeline -> Encoded Byte Stream -> Network

> **Inbound Flow (Receiving):**
> Network -> Raw Byte Stream -> Netty Channel Pipeline -> **SetServerAccess.deserialize()** -> Packet Handler -> Application Logic

