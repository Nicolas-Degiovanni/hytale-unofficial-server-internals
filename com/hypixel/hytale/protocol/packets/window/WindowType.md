---
description: Architectural reference for WindowType
---

# WindowType

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum WindowType {
```

## Architecture & Concepts
The WindowType enum serves as a type-safe, human-readable representation of the various user interface windows that can be opened in the game. It is a fundamental component of the client-server communication protocol, specifically for packets related to UI and inventory management.

Its primary role is to translate a raw integer identifier, received from the network, into a well-defined constant. This pattern eliminates the use of "magic numbers" throughout the codebase, improving readability and reducing the risk of errors. For example, instead of checking if a window ID is 3, the system can perform a more explicit check against WindowType.DiagramCrafting.

This enum acts as a data contract between the client and server. Any changes to these values must be synchronized across both applications to prevent protocol desynchronization.

### Lifecycle & Ownership
- **Creation:** All enum constants are instantiated by the Java Virtual Machine during class loading. This process is automatic and occurs before any application code explicitly references the WindowType class.
- **Scope:** The enum constants are static, final, and exist for the entire lifetime of the application. They are globally accessible once the class is loaded.
- **Destruction:** The constants are garbage collected when the JVM shuts down. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** WindowType is immutable. Each enum constant holds a single, final integer field representing its protocol value. The static VALUES array, used for efficient lookups, is also final and populated only once at class initialization.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature and the guarantees provided by the JVM for enum initialization ensure that it can be safely accessed from any thread without synchronization. The fromValue method is a pure function operating on immutable data and is safe for concurrent use.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant, used for serialization. |
| fromValue(int value) | WindowType | O(1) | Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The primary integration point for WindowType is within the network protocol layer during packet deserialization. A handler will read an integer from the network buffer and use the fromValue static method to resolve it to a specific type.

```java
// Example within a packet handler
int rawWindowType = buffer.readVarInt();

// The fromValue method is the designated factory for converting
// a network ID into a usable object.
WindowType type = WindowType.fromValue(rawWindowType);

// The UI system can now use the type-safe enum to open the correct window
uiManager.openWindow(type);
```

### Anti-Patterns (Do NOT do this)
- **Using Raw Integer Values:** Avoid comparing the integer value directly. This defeats the purpose of a type-safe enum and makes the code brittle.
  - **BAD:** `if (window.getType().getValue() == 3)`
  - **GOOD:** `if (window.getType() == WindowType.DiagramCrafting)`
- **Relying on Ordinal:** Never use the built-in `ordinal()` method for serialization or logic. The declaration order of enums can change, which would break the protocol. The custom `getValue()` method is the only supported way to get the protocol identifier.
- **Ignoring ProtocolException:** The ProtocolException thrown by fromValue indicates a critical error, such as data corruption or a client/server version mismatch. This exception must be handled, typically by disconnecting the client to prevent further state corruption.

## Data Pipeline
WindowType acts as a simple but critical transformation step in the data flow from the network to the UI system.

> Flow:
> Network Packet -> Protocol Deserializer -> **WindowType.fromValue()** -> UI Manager -> UI Window Instantiation

