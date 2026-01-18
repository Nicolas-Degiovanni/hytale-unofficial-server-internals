---
description: Architectural reference for WriteUpdateType
---

# WriteUpdateType

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Enumeration / Static Utility

## Definition
```java
// Signature
public enum WriteUpdateType {
```

## Architecture & Concepts
The WriteUpdateType enum defines a fixed set of operations for asset manipulation within the Hytale protocol. It serves as a type-safe, low-overhead data contract for serializing commands between the client and server.

Instead of transmitting human-readable strings like "Add" or "Remove", the protocol uses the integer value associated with each enum constant. This minimizes packet size and reduces parsing overhead. The static factory method, fromValue, is the designated entry point for deserialization, converting a raw integer from a network stream back into a strongly-typed object. This method also acts as a validation layer, throwing a ProtocolException for any undefined integer values, which prevents the system from entering an invalid state due to malformed or malicious packets.

## Lifecycle & Ownership
- **Creation:** Instantiated by the Java Virtual Machine (JVM) during class loading. The enum constants (Add, Update, Remove) are created once and exist as singleton instances.
- **Scope:** Application-wide. The constants persist for the entire lifetime of the application.
- **Destruction:** Managed by the JVM. The enum and its constants are unloaded only when the defining class loader is garbage collected. User code should never attempt to manage their lifecycle.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final, private integer value that is set at compile time. The class itself is stateless.
- **Thread Safety:** Inherently thread-safe. The JVM guarantees that enum constants are initialized in a thread-safe manner. They can be safely accessed and read from any thread without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable, integer-based protocol identifier for the enum constant. |
| fromValue(int value) | WriteUpdateType | O(1) | Static factory method to deserialize an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is not a valid identifier. |

## Integration Patterns

### Standard Usage
This enum is primarily used during packet serialization and deserialization. When writing a packet, use getValue to get the network-safe integer. When reading, use fromValue to validate and convert the integer back to a type-safe constant.

```java
// Writing an "Add" operation to a buffer
int operationId = WriteUpdateType.Add.getValue();
buffer.writeInt(operationId);

// Reading an operation from a buffer
int receivedId = buffer.readInt();
WriteUpdateType operation = WriteUpdateType.fromValue(receivedId);

switch (operation) {
    case Add:
        // Handle asset addition
        break;
    case Update:
        // Handle asset update
        break;
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Using ordinal():** Do not use the built-in `ordinal()` method for serialization. The integer values are explicitly defined via `getValue()` to ensure the protocol remains stable even if the declaration order of the enum constants changes. Using `ordinal()` creates a brittle contract that can easily break.
- **Ignoring ProtocolException:** The `fromValue` method can throw an exception. This is a critical part of the protocol's integrity. Failure to catch this exception can lead to unhandled connection termination or state corruption when processing invalid data.

## Data Pipeline
The WriteUpdateType enum is a key component in the data translation layer between high-level game logic and the low-level network byte stream.

> **Outbound Flow (Serialization):**
> Game Logic Command -> **WriteUpdateType.Add** -> `getValue()` -> `int (0)` -> Network Buffer

> **Inbound Flow (Deserialization):**
> Network Buffer -> `int (0)` -> `fromValue(0)` -> **WriteUpdateType.Add** -> Game Logic Handler

