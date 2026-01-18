---
description: Architectural reference for CustomUICommandType
---

# CustomUICommandType

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Static Enum

## Definition
```java
// Signature
public enum CustomUICommandType {
```

## Architecture & Concepts
CustomUICommandType is a foundational enumeration within the Hytale network protocol, specifically for the server-driven user interface system. It serves as a type-safe and constrained vocabulary for UI manipulation commands sent from the server to the client.

Each enum constant, such as Append or Remove, represents a distinct, high-level operation that the client's UI framework must perform. By using an integer-backed enum, the protocol achieves network efficiency while maintaining code readability and type safety on both the client and server.

This component acts as a contract between the server's intent and the client's UI rendering engine. The server serializes a command into its integer representation, and the client deserializes it back into the corresponding enum constant. This decouples the packet parsing layer from the UI implementation; the parser's only responsibility is to validate and provide the correct CustomUICommandType, which is then consumed by a dedicated UI handler.

### Lifecycle & Ownership
- **Creation:** All instances of this enum are created and initialized by the Java Virtual Machine during class loading. They are static, final constants. The static factory method fromValue does not create new instances but rather returns one of the pre-existing ones.
- **Scope:** Application-scoped. The enum constants exist for the entire lifetime of the client or server process.
- **Destruction:** Instances are managed by the JVM and are reclaimed only upon application shutdown. There is no manual memory management.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant holds a final private integer field, which is set at compile time and cannot be modified.
- **Thread Safety:** Inherently thread-safe. As immutable singletons, instances of CustomUICommandType can be safely passed between and accessed by multiple threads without any need for external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value of the enum constant. Intended for serialization to the network stream. |
| fromValue(int value) | CustomUICommandType | O(1) | A static factory method for deserialization. Translates an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used at the boundary between network packet processing and UI system logic. The typical pattern involves reading an integer from the network buffer, converting it to a command, and then using a switch statement to execute the appropriate logic.

```java
// In a packet handler or UI processing service
int commandId = buffer.readVarInt();
CustomUICommandType command = CustomUICommandType.fromValue(commandId);

switch (command) {
    case Append:
        uiManager.appendElement(...);
        break;
    case Remove:
        uiManager.removeElement(...);
        break;
    // ... other cases
}
```

### Anti-Patterns (Do NOT do this)
- **Using Magic Numbers:** Avoid using raw integer values (0, 1, 4, etc.) directly in application logic. This creates brittle code that is hard to read and maintain. The purpose of the enum is to provide symbolic constants. The getValue and fromValue methods should only be used at the serialization boundary.
- **Ignoring Exceptions:** The fromValue method can throw a ProtocolException. Failure to handle this exception can lead to a client disconnect or a corrupted state. This is a critical validation step for network data.

## Data Pipeline
The CustomUICommandType is a key transformation point in the flow of UI data from the server to the client's screen.

> Flow:
> Server UI Logic -> Network Packet (Integer) -> Client Packet Decoder -> **CustomUICommandType.fromValue()** -> UI Command Handler -> UI State Update -> Render Engine

