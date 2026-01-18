---
description: Architectural reference for PasswordResponse
---

# PasswordResponse

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Data Transfer Object (Transient)

## Definition
```java
// Signature
public class PasswordResponse implements Packet {
```

## Architecture & Concepts
The PasswordResponse class is a specialized Data Transfer Object (DTO) that represents a single, specific message within Hytale's authentication protocol. It serves as a structured container for the client's password hash, which is sent to the server for verification during the login sequence.

This class is a fundamental component of the network protocol layer. It does not contain any business logic; its sole responsibility is to model the data for packet ID 15. The design emphasizes performance and strict adherence to the binary protocol format, featuring hand-written serialization and deserialization logic that operates directly on Netty's ByteBuf. This approach avoids the overhead of reflection-based serialization frameworks and provides granular control over byte-level layout, which is critical for a high-performance game network.

Its role is to act as an immutable contract between the client and server for this specific step of the authentication process.

### Lifecycle & Ownership
- **Creation:** An instance is created under two primary circumstances:
    1. **Deserialization:** The network protocol layer instantiates a PasswordResponse object via the static *deserialize* factory method when an incoming data stream corresponding to packet ID 15 is received.
    2. **Serialization:** The client-side authentication logic instantiates a PasswordResponse object and populates it with the user's password hash before sending it to the server.
- **Scope:** The object's lifetime is extremely short. It is scoped to the immediate handling of a single network event. It exists only long enough to be read from a network buffer and passed to a handler, or to be constructed and written to a network buffer.
- **Destruction:** The object is eligible for garbage collection immediately after the network event handler completes its execution or after the data has been successfully written to the network socket. There is no persistent state.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of a single nullable byte array, *hash*. The state is simple and directly represents the data payload of the network packet. The maximum size of the hash is strictly enforced at 64 bytes.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, populated, and processed within the confines of a single thread, typically a Netty I/O worker thread. Accessing or modifying a PasswordResponse instance from multiple threads without external synchronization will result in undefined behavior and data corruption. No internal locking mechanisms are provided.

## API Surface
The public API is focused entirely on protocol-level operations: serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | PasswordResponse | O(N) | Static factory. Reads from a ByteBuf to construct a new instance. N is the length of the hash. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the protocol specification. N is the length of the hash. |
| computeSize() | int | O(1) | Calculates the exact number of bytes this packet will occupy on the wire. Essential for buffer pre-allocation. |
| validateStructure(ByteBuf, int) | ValidationResult | O(N) | Performs a read-only check to ensure the data in the buffer is a valid representation of this packet without full deserialization. N is the length of the hash. |
| clone() | PasswordResponse | O(N) | Creates a deep copy of the packet, including a new copy of the internal hash byte array. N is the length of the hash. |

## Integration Patterns

### Standard Usage
This class is almost exclusively used by the core network and authentication systems. A developer would typically interact with it inside a packet handler.

```java
// Example of a handler receiving the packet
public void handlePasswordResponse(PasswordResponse packet) {
    byte[] clientHash = packet.hash;
    if (authenticationService.verifyHash(clientHash)) {
        // Proceed with login
    } else {
        // Disconnect client
    }
}

// Example of the client sending the packet
byte[] userHash = computeUserPasswordHash();
PasswordResponse response = new PasswordResponse(userHash);
connection.sendPacket(response);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not modify and re-send a packet instance after it has been passed to the network layer. Packet objects are cheap; create a new one for each message.
- **Cross-Thread Sharing:** Do not pass a PasswordResponse instance from a network thread to a worker thread without creating a defensive copy (using *clone*) or extracting the data into a thread-safe structure. Direct sharing can lead to race conditions.
- **Ignoring Size Limits:** Do not attempt to construct a PasswordResponse with a hash larger than 64 bytes. The *serialize* method will throw a ProtocolException.

## Data Pipeline
The PasswordResponse class is a data structure that moves through the network pipeline. It is instantiated at the pipeline's edge on both the sending and receiving ends.

> **Flow (Client to Server):**
> Client Authentication Logic → **PasswordResponse (Instantiation)** → Protocol Serialization Engine → Netty ByteBuf → Network Socket

> **Flow (Server from Client):**
> Network Socket → Netty ByteBuf → Protocol Deserialization Engine → **PasswordResponse (Instantiation)** → Server Authentication Handler

