---
description: Architectural reference for ProtocolSettings
---

# ProtocolSettings

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
// Signature
public final class ProtocolSettings {
```

## Architecture & Concepts
The ProtocolSettings class is a static manifest containing compile-time constants that define the canonical properties of the network protocol. It is not a dynamic component but rather a single source of truth for protocol versioning and metadata. Its primary function is to enable a robust version-checking mechanism between the client and server during the initial connection handshake.

The core of this mechanism is the **PROTOCOL_HASH**, a SHA-256 hash likely generated from the protocol's definition files. Any modification to a packet, struct, or enum definition will result in a different hash. This ensures that clients and servers built against different protocol schemas cannot communicate, preventing critical deserialization errors and state corruption.

Other constants like PROTOCOL_VERSION and component counts (PACKET_COUNT, STRUCT_COUNT) serve as supplementary metadata for validation, debugging, or pre-allocating data structures within the network layer.

### Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its private constructor enforces a pure-static access pattern. The Java Virtual Machine loads the class and its static members upon first reference.
- **Scope:** Application-scoped. The constants are available globally for the entire lifetime of the application process.
- **Destruction:** The class and its data are unloaded when the JVM shuts down. No manual resource management is required.

## Internal State & Concurrency
- **State:** ProtocolSettings is stateless and deeply immutable. All fields are declared `public static final`, making them compile-time constants that cannot be changed at runtime.
- **Thread Safety:** This class is inherently thread-safe. As it contains only immutable static data and pure static functions, it can be safely accessed from any thread without synchronization primitives.

## API Surface
The API consists entirely of public static constants and methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PROTOCOL_HASH | String | O(1) | The canonical SHA-256 hash of the protocol definition. Used for version validation. |
| PROTOCOL_VERSION | int | O(1) | The major version number of the network protocol. |
| PACKET_COUNT | int | O(1) | The total number of unique packet types defined in the protocol. |
| STRUCT_COUNT | int | O(1) | The total number of unique struct types defined in the protocol. |
| ENUM_COUNT | int | O(1) | The total number of unique enum types defined in the protocol. |
| MAX_PACKET_SIZE | int | O(1) | The maximum permissible size in bytes for a single network packet. |
| validateHash(hash) | boolean | O(1) | Compares a provided hash against the canonical PROTOCOL_HASH. Returns true on match. |

## Integration Patterns

### Standard Usage
The primary integration point for this class is within the server's connection handling logic. During the initial handshake, the server must validate the client's protocol hash to ensure compatibility before proceeding.

```java
// Example: Server-side connection handler
public void onClientHandshake(HandshakePacket packet) {
    String clientHash = packet.getProtocolHash();

    if (!ProtocolSettings.validateHash(clientHash)) {
        // Hashes do not match. The client is incompatible.
        // Terminate the connection immediately with a version mismatch error.
        connection.disconnect("Incompatible client protocol.");
        return;
    }

    // Hashes match. Proceed with the connection lifecycle.
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance of ProtocolSettings. The class is not designed for instantiation, and the private constructor will prevent it. All access must be static.
- **Runtime Modification:** Do not attempt to modify the constant fields using reflection. This would violate the protocol contract, lead to unpredictable behavior, and break compatibility guarantees.
- **Bypassing Validation:** The `validateHash` check is a critical security and stability feature. Bypassing this check or hardcoding a `true` return value in a development environment risks introducing subtle and difficult-to-diagnose bugs related to data corruption.

## Data Pipeline
ProtocolSettings acts as a gatekeeper at the very beginning of the network data pipeline, not as a processing stage within it. It is used to make a binary decision on whether to allow a connection to be established.

> Flow:
> Client Handshake Packet -> Server Network Layer -> **ProtocolSettings.validateHash()** -> [Connection Accepted | Connection Rejected]

