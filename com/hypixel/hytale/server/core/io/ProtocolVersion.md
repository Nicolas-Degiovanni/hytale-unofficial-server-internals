---
description: Architectural reference for ProtocolVersion
---

# ProtocolVersion

**Package:** com.hypixel.hytale.server.core.io
**Type:** Transient (Value Object)

## Definition
```java
// Signature
public class ProtocolVersion {
```

## Architecture & Concepts
The ProtocolVersion class is an immutable value object designed to provide a type-safe representation of a client or server network protocol identifier. Its primary role is to encapsulate the protocol hash, preventing the "primitive obsession" anti-pattern where a raw String might be used.

This component is fundamental to the initial client-server handshake process. By wrapping the protocol hash in a dedicated type, the system can enforce version compatibility checks with greater clarity and safety. Its identity is defined entirely by its value; two ProtocolVersion instances with the same hash are considered equal.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand, typically by network layer components during the connection handshake or by configuration loaders when the server boots. It is not a managed service.
- **Scope:** Ephemeral and short-lived. A ProtocolVersion object exists only as long as it is needed for a comparison or to be stored within a session or configuration object.
- **Destruction:** As a simple heap-allocated object with no external resources, it is reclaimed by the standard Java garbage collector once all references are released.

## Internal State & Concurrency
- **State:** Strictly immutable. The internal hash field is final and is assigned only once during construction. The object's state cannot be changed after creation.
- **Thread Safety:** Inherently thread-safe due to its immutability. Instances can be safely shared and read by multiple threads without any external synchronization or locking mechanisms.

## API Surface
The public contract is minimal, focusing on value retrieval and comparison.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHash() | String | O(1) | Returns the raw protocol hash string. |
| equals(Object o) | boolean | O(N) | Performs a value-based comparison against another object. N is the length of the hash string. |

## Integration Patterns

### Standard Usage
The class should be used to compare the server's protocol version with that of a connecting client to determine compatibility.

```java
// How a developer should normally use this
ProtocolVersion serverVersion = config.getProtocolVersion();
ProtocolVersion clientVersion = new ProtocolVersion(packet.getProtocolHash());

if (!serverVersion.equals(clientVersion)) {
    // Reject connection due to protocol mismatch
    connection.reject("Incompatible client version.");
}
```

### Anti-Patterns (Do NOT do this)
- **Null Hash:** Do not construct a ProtocolVersion with a null hash. While the internal `equals` and `hashCode` methods are null-safe, a null hash represents an indeterminate and invalid protocol state that will cause logic errors upstream.
- **String Comparison:** Avoid extracting the raw hash for comparison. Rely on the built-in `equals` method to ensure correct, type-safe comparison logic.

```java
// BAD: Bypasses the type safety of the object
if (serverVersion.getHash().equals(clientVersion.getHash())) {
    // ...
}

// GOOD: Uses the intended comparison method
if (serverVersion.equals(clientVersion)) {
    // ...
}
```

## Data Pipeline
ProtocolVersion acts as a data container within the network connection pipeline. It does not process data itself but is a critical piece of information used in routing and validation logic.

> Flow:
> Client Connection Packet -> Network Packet Decoder -> **ProtocolVersion (instantiated)** -> Handshake Service (version comparison) -> Connection Accepted or Rejected

