---
description: Architectural reference for PongType
---

# PongType

**Package:** com.hypixel.hytale.protocol.packets.connection
**Type:** Enumeration

## Definition
```java
// Signature
public enum PongType {
```

## Architecture & Concepts
PongType is a type-safe enumeration that defines the distinct categories of "pong" responses within the Hytale network protocol. Its primary function is to provide a robust and readable abstraction over the raw integer identifiers used during network serialization.

This enum is a foundational element of the connection-level packet system. It ensures that both the client and server have a shared, compile-time understanding of the pong message variants. By mapping specific integer values (0 for Raw, 1 for Direct, 2 for Tick) to named constants, it eliminates the use of "magic numbers" in the protocol handling logic.

The inclusion of the static factory method, fromValue, centralizes the deserialization logic and provides a single point of validation. Any attempt to deserialize an undefined or out-of-bounds integer value results in a controlled ProtocolException, immediately halting the processing of a malformed packet and preventing state corruption.

### Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are effectively static, singleton instances managed by the runtime.
-   **Scope:** Application-wide. The PongType constants exist for the entire lifetime of the application once the class is loaded.
-   **Destruction:** The constants are garbage collected by the JVM during application shutdown. No manual memory management is required or possible.

## Internal State & Concurrency
-   **State:** Immutable. The internal state of each enum constant, specifically its integer value, is final and assigned at compile time. The set of available constants is fixed.
-   **Thread Safety:** Inherently thread-safe. As an immutable, statically-defined type, PongType can be safely accessed and referenced from any thread without external synchronization. The fromValue and getValue methods are pure functions and are also thread-safe.

## API Surface
The primary API consists of the constants themselves and the serialization/deserialization methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Raw | PongType | O(1) | Constant representing a raw pong response. |
| Direct | PongType | O(1) | Constant representing a direct pong response. |
| Tick | PongType | O(1) | Constant representing a tick-synchronized pong response. |
| fromValue(int value) | static PongType | O(1) | Deserializes an integer into a PongType. Throws ProtocolException on invalid input. |
| getValue() | int | O(1) | Serializes the enum constant to its integer representation for network transmission. |

## Integration Patterns

### Standard Usage
PongType is used during the serialization and deserialization of network packets that involve a pong mechanism, such as keep-alive or latency checks.

```java
// Serialization: Writing the type to a network buffer
PongType typeToSend = PongType.Tick;
int protocolValue = typeToSend.getValue();
networkBuffer.writeInt(protocolValue);

// Deserialization: Reading the type from a network buffer
try {
    int receivedValue = networkBuffer.readInt();
    PongType receivedType = PongType.fromValue(receivedValue);
    // ... handle logic based on receivedType
} catch (ProtocolException e) {
    // Disconnect client for sending invalid data
    connection.disconnect("Invalid PongType value.");
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Ordinals for Serialization:** **NEVER** use the built-in `ordinal()` method for serialization. The protocol relies on the explicit integer values defined in the constructor. The order of enum constant declaration is not guaranteed to be stable and using `ordinal()` will break protocol compatibility if the order is ever changed. Always use `getValue()` for serialization and `fromValue()` for deserialization.
-   **Ignoring ProtocolException:** The `fromValue` method is a critical validation boundary. Failure to catch the ProtocolException it throws for invalid data will result in an unhandled exception, potentially crashing the network thread. Always wrap calls to `fromValue` in a try-catch block and handle the exception, typically by terminating the offending connection.

## Data Pipeline
PongType acts as a data contract within the network serialization pipeline. It does not process data itself but ensures data integrity as it flows through the system.

> **Serialization Flow:**
> Game Logic -> Pong Packet (contains **PongType**) -> Packet Serializer -> `getValue()` -> Network Stream (as integer)

> **Deserialization Flow:**
> Network Stream (as integer) -> Packet Deserializer -> `fromValue()` -> **PongType** -> Pong Packet -> Game Logic

