---
description: Architectural reference for PasswordRejected
---

# PasswordRejected

**Package:** com.hypixel.hytale.protocol.packets.auth
**Type:** Data Transfer Object

## Definition
```java
// Signature
public class PasswordRejected implements Packet {
```

## Architecture & Concepts
The PasswordRejected class is a Data Transfer Object (DTO) that represents a specific, concrete message within the Hytale authentication protocol. It is not a service or manager; it is a passive data structure whose sole purpose is to carry state from the server to the client.

This packet is sent exclusively by the server in response to a client's failed password submission. Its design serves two primary functions:
1.  **Rejection Notification:** Unambiguously informs the client that the provided credentials were incorrect.
2.  **Retry Mechanism:** Provides a new cryptographic challenge and a remaining attempt counter, enabling a secure, stateful retry loop without requiring a full connection reset.

Architecturally, this class acts as a serialization boundary contract between the raw byte stream managed by the Netty network layer and the high-level authentication state machine on the client. The static methods for serialization and deserialization are the key components that enforce this contract, ensuring byte-level compatibility between server and client implementations.

## Lifecycle & Ownership
-   **Creation:**
    -   **Server-Side:** Instantiated by the authentication service when a client's password verification fails. The object is populated with a new challenge and the updated attempt count. It is then passed to the network pipeline for serialization.
    -   **Client-Side:** Instantiated by the protocol decoding layer. When an incoming data frame is identified with a Packet ID of 17, the static `deserialize` method is invoked to construct a PasswordRejected object from the raw Netty ByteBuf.

-   **Scope:** Extremely short-lived and transient. An instance of this class exists only for the brief moment it takes to be passed from the network decoder to the authentication event handler. It represents a point-in-time event, not persistent state.

-   **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by the client's authentication logic. No long-term references are maintained.

## Internal State & Concurrency
-   **State:** The class holds mutable state in its public fields, `newChallenge` and `attemptsRemaining`. However, by convention and design, it should be treated as an immutable record after deserialization. Its purpose is to transport a snapshot of data, not to have its state managed over time.

-   **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, typically a Netty I/O worker thread. All operations, from deserialization to consumption by an event handler, are expected to occur synchronously on the same thread. Passing instances of this class between threads requires external synchronization, which would be a violation of the protocol's intended design.

## API Surface
The primary API is static and focused on protocol-level operations. Instance methods are simple data accessors or part of the Packet contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static PasswordRejected | O(N) | Constructs a PasswordRejected object from a raw ByteBuf. N is the length of the challenge. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a ByteBuf for network transmission. N is the length of the challenge. |
| validateStructure(buf, offset) | static ValidationResult | O(1) | Performs a lightweight check to see if a buffer could contain a valid packet without full deserialization. Critical for security and preventing parsing exploits. |
| computeSize() | int | O(1) | Calculates the exact number of bytes the serialized packet will occupy. |
| getId() | int | O(1) | Returns the static packet identifier, which is 17. |

## Integration Patterns

### Standard Usage
The PasswordRejected packet is handled by the client's network event loop. The decoder identifies the packet ID and uses the static `deserialize` method to create the object, which is then passed to a higher-level handler to update the UI and authentication state.

```java
// Executed within a client-side network handler
// byteBuf contains the raw data from the server

if (packetId == PasswordRejected.PACKET_ID) {
    PasswordRejected rejection = PasswordRejected.deserialize(byteBuf, offset);

    // Dispatch to the authentication controller or event bus
    AuthenticationController.handlePasswordRejected(
        rejection.attemptsRemaining,
        rejection.newChallenge
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Client-Side Instantiation:** Do not use `new PasswordRejected()` on the client. This packet is a server-to-client message. Manually creating it on the client serves no purpose and indicates a misunderstanding of the authentication flow.
-   **State Modification:** Do not modify the fields of a deserialized packet. Treat it as a read-only event record. Altering `attemptsRemaining` or `newChallenge` on the client has no effect on the server and will corrupt the client's state.
-   **Asynchronous Processing:** Do not hand off a deserialized packet to another thread without deep-copying its data. The underlying ByteBuf may be recycled by Netty, leading to data corruption or memory access violations.

## Data Pipeline
The data flow for this packet is unidirectional from server to client.

> **Server Flow:**
> Authentication Service (Login Failure) -> `new PasswordRejected()` -> Protocol Encoder -> Netty Channel -> TCP/IP Stack

> **Client Flow:**
> TCP/IP Stack -> Netty Channel -> Protocol Decoder -> **PasswordRejected.deserialize()** -> Authentication Event Handler -> UI Update ("Password Incorrect")

