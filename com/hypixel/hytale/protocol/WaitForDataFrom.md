---
description: Architectural reference for WaitForDataFrom
---

# WaitForDataFrom

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum WaitForDataFrom {
```

## Architecture & Concepts
The WaitForDataFrom enumeration is a fundamental component of the Hytale network protocol's state management system. It is not merely a data container but a state machine directive that governs the flow of communication between the client and server.

At its core, this enum enforces a turn-based communication model for specific protocol sequences. When a connection handler is in a state governed by this enum, it explicitly defines which party—the Client or the Server—is expected to transmit data next. This mechanism is critical for preventing protocol deadlocks, race conditions, and ambiguous connection states during complex handshakes or multi-step operations.

For example, after a client sends an authentication request, the connection's state might transition to require `WaitForDataFrom.Server`. This instructs the protocol handler to pause sending further data and listen exclusively for a response from the server, ensuring a strict, predictable sequence of events. The `None` state typically signifies the end of a sequence or a state where either party can initiate communication.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances (Client, Server, None) are compile-time constants. They are instantiated automatically by the Java Virtual Machine during class loading. There is no dynamic or manual instantiation.
- **Scope:** Application-wide. The enum constants exist for the entire lifetime of the JVM process once the class is loaded.
- **Destruction:** The instances are managed by the JVM and are only eligible for garbage collection upon application shutdown. Manual cleanup is neither possible nor necessary.

## Internal State & Concurrency
- **State:** **Immutable**. The enum constants and their associated integer values are fixed at compile time and cannot be modified at runtime. The internal `VALUES` array is a static final cache, making it effectively immutable after its one-time initialization.
- **Thread Safety:** **Inherently thread-safe**. Due to their immutable and static nature, instances of WaitForDataFrom can be safely accessed and shared across any number of threads without requiring locks or other synchronization primitives. All associated methods are pure functions and are also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the enum constant, primarily used for network serialization. |
| fromValue(int value) | WaitForDataFrom | O(1) | A static factory method for deserialization. Converts a raw integer from a network stream into its corresponding enum constant. **Warning:** Throws ProtocolException if the integer value is out of bounds, indicating corrupt or invalid data. |

## Integration Patterns

### Standard Usage
This enum is intended to be used within protocol state machines to direct logic, typically inside a switch statement. The `fromValue` and `getValue` methods facilitate the serialization boundary.

```java
// Example: Deserializing and processing a state update
int rawState = networkBuffer.readVarInt();
WaitForDataFrom nextStep = WaitForDataFrom.fromValue(rawState);

switch (nextStep) {
    case Client:
        // Prepare to receive a packet from the client
        connection.listenForClientPacket();
        break;
    case Server:
        // Prepare to receive a packet from the server
        connection.listenForServerPacket();
        break;
    case None:
        // The sequence is complete, transition to an idle state
        connection.enterIdleState();
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Magic Numbers:** Never use the raw integer values (0, 1, 2) in application logic. Relying on these values defeats the purpose of type safety and creates brittle code that is difficult to maintain. Always compare against the enum constants directly (e.g., `if (state == WaitForDataFrom.Client)`).
- **Unhandled Exceptions:** Failure to catch the ProtocolException from `fromValue` can lead to unhandled exceptions that may terminate a connection handler thread. Always wrap calls to `fromValue` in a try-catch block when processing untrusted network input.

## Data Pipeline
WaitForDataFrom acts as a deserialization and state transition gate within the protocol layer. It translates a primitive integer from a raw data stream into a high-level, type-safe state directive that the connection manager can act upon.

> Flow:
> Network Byte Stream -> Protocol Decoder -> `fromValue(intValue)` -> **WaitForDataFrom** instance -> Protocol State Machine -> Connection Handler Action

