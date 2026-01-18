---
description: Architectural reference for TransportType
---

# TransportType

**Package:** com.hypixel.hytale.server.core.io.transport
**Type:** Utility

## Definition
```java
// Signature
public enum TransportType {
```

## Architecture & Concepts
TransportType is a type-safe enumeration that defines the supported network transport protocols for the Hytale server. It serves as a foundational configuration primitive within the server's networking layer. Its primary architectural role is to decouple the server's core logic from specific transport implementations (e.g., TCP, QUIC).

By using this enum, the server can be configured at startup to select a specific network stack. This allows for future expansion with new protocols without requiring changes to the core server bootstrap or connection management logic. It acts as a selector or a factory key, enabling the `TransportManager` to instantiate the correct protocol-specific handlers and channel initializers.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated automatically by the Java Virtual Machine (JVM) when the TransportType class is loaded. This process is managed entirely by the JVM's ClassLoader.
- **Scope:** As static final constants, the instances (TCP, QUIC) exist for the entire lifetime of the application. They are globally accessible and effectively singletons.
- **Destruction:** The enum instances are garbage collected only when the defining ClassLoader is unloaded, which typically occurs only upon application shutdown.

## Internal State & Concurrency
- **State:** TransportType is fundamentally immutable. Its instances (TCP, QUIC) are constants with no mutable state.
- **Thread Safety:** This enum is inherently thread-safe. As an immutable, globally accessible set of constants, it can be safely referenced from any thread without synchronization. This is a core guarantee of the Java enum pattern.

## API Surface
The primary API consists of its defined constants. Standard enum methods like `values()` and `valueOf(String)` are also available but should be used with caution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TCP | TransportType | O(1) | Represents the Transmission Control Protocol. |
| QUIC | TransportType | O(1) | Represents the QUIC protocol. |

## Integration Patterns

### Standard Usage
This enum is intended to be used as a configuration value, typically read from a server properties file and used to initialize the network stack.

```java
// Example: Reading configuration and selecting a transport
TransportType configuredType = serverConfig.getTransportType(); // e.g., returns TransportType.TCP

if (configuredType == TransportType.TCP) {
    // Initialize TCP-specific network stack
} else if (configuredType == TransportType.QUIC) {
    // Initialize QUIC-specific network stack
}
```

### Anti-Patterns (Do NOT do this)
- **String Comparison:** Do not compare the string representation of the enum. This is inefficient, error-prone, and defeats the purpose of type safety.
  - **BAD:** `if (configuredType.name().equals("TCP"))`
  - **GOOD:** `if (configuredType == TransportType.TCP)`
- **Unsafe Deserialization:** Avoid using `TransportType.valueOf(String)` with unvalidated, user-provided strings. This can throw an `IllegalArgumentException` if the string does not match a defined constant, potentially causing an unhandled exception path.

## Data Pipeline
TransportType does not process data itself; it is a configuration value that dictates which data processing pipeline is constructed at server startup.

> Flow:
> Server Configuration File -> Config Loader -> **TransportType** -> TransportManager -> (TCP Pipeline OR QUIC Pipeline)

