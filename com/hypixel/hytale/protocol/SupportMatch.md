---
description: Architectural reference for SupportMatch
---

# SupportMatch

**Package:** com.hypixel.hytale.protocol
**Type:** Utility

## Definition
```java
public enum SupportMatch {
```

## Architecture & Concepts
The SupportMatch enum is a foundational component of the Hytale network protocol layer, specifically designed for feature negotiation during the client-server handshake. It provides a type-safe, self-documenting representation of the three possible states of feature compatibility: *Ignored*, *Required*, or *Disallowed*.

This enum is critical for protocol versioning and capability exchange. When a client connects, the server can query its support for various features (e.g., a new compression algorithm, an updated entity format). The client responds with a SupportMatch value for each feature. This system allows for graceful degradation and forward compatibility, preventing clients or servers with mismatched feature sets from causing protocol errors.

Its primary architectural role is to eliminate "magic numbers" from the network deserialization and feature-handling logic, replacing ambiguous integers with strongly-typed, immutable constants. The static factory method, fromValue, acts as a validation and deserialization gateway, ensuring that any integer read from a network stream corresponds to a valid, known state.

## Lifecycle & Ownership
- **Creation:** All enum constants (Ignored, Required, Disallowed) are instantiated by the Java Virtual Machine during class loading. This process is guaranteed to happen only once. The internal VALUES array is also populated at this time as a static initializer.
- **Scope:** Application-wide. As a static enumeration, its constants persist for the entire lifetime of the JVM process.
- **Destruction:** The enum and its constants are unloaded when the application's class loader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final primitive integer. The internal state cannot be modified after its initial creation by the JVM. The static VALUES array is populated once and is never modified thereafter.
- **Thread Safety:** This class is inherently thread-safe. Enum constants are global singletons, and their immutability guarantees that they can be safely accessed and passed between any number of threads without synchronization or locking mechanisms. All methods are re-entrant and free of side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value corresponding to the enum constant, suitable for serialization into a network packet. |
| fromValue(int value) | static SupportMatch | O(1) | Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the integer is out of the defined range. |

## Integration Patterns

### Standard Usage
The fromValue method is the designated entry point for converting raw network data into a validated SupportMatch state. Business logic should then use a switch statement to handle each case explicitly.

```java
// Deserialize the feature support value from a network buffer
int featureSupportValue = buffer.readVarInt();

try {
    SupportMatch match = SupportMatch.fromValue(featureSupportValue);

    // Act based on the negotiated feature support
    switch (match) {
        case Required:
            // Enable the feature; connection fails if not possible
            enableRequiredFeature();
            break;
        case Ignored:
            // Feature is optional, proceed without it
            log.info("Optional feature is ignored by the remote peer.");
            break;
        case Disallowed:
            // The remote peer forbids this feature, terminate connection
            throw new ConnectionRejectedException("Feature explicitly disallowed.");
    }
} catch (ProtocolException e) {
    // Handle invalid data from the network
    disconnect("Invalid feature support value received.");
}
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integers:** Do not compare against raw integer values in your application logic (e.g., `if (featureSupportValue == 1)`). This defeats the purpose of the enum, re-introduces magic numbers, and makes the code brittle and hard to read.
- **Ignoring ProtocolException:** The fromValue method is a critical validation boundary. Failure to catch the ProtocolException it throws can lead to unhandled exceptions and abrupt connection termination when malformed or malicious data is received.

## Data Pipeline
The SupportMatch enum serves as a deserialization and validation endpoint for a single integer field within a larger network packet.

> Flow:
> Network Packet (int) -> Protocol Deserializer -> **SupportMatch.fromValue(int)** -> SupportMatch (Enum Constant) -> Feature Negotiation Logic

