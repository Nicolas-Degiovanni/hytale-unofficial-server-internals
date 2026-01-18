---
description: Architectural reference for AssetEditorPopupNotificationType
---

# AssetEditorPopupNotificationType

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Utility

## Definition
```java
// Signature
public enum AssetEditorPopupNotificationType {
```

## Architecture & Concepts
AssetEditorPopupNotificationType is a static enumeration that defines a constrained set of notification severities for the in-game Asset Editor. It serves as a critical component of the client-server protocol, ensuring type-safe and standardized communication for UI feedback.

This enum acts as a contract between the server, which may trigger notifications based on asset validation or processing results, and the client, which must render the appropriate UI popup. By mapping symbolic names like *Success* or *Error* to fixed integer values, it eliminates the use of "magic numbers" in the network stream, improving protocol readability and maintainability. Its primary role is data serialization and deserialization within the packet layer.

### Lifecycle & Ownership
- **Creation:** All instances (Info, Success, Error, Warning) are instantiated by the JVM during class loading. They are compile-time constants and exist as effective singletons.
- **Scope:** Application-level. The enum constants persist for the entire lifetime of the application.
- **Destruction:** The constants are garbage collected only when the defining class loader is unloaded, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** Immutable. Each enum constant encapsulates a final, private integer value that is set at creation and cannot be modified. The static VALUES array is also effectively immutable.
- **Thread Safety:** Inherently thread-safe. As immutable, globally accessible singletons, these constants can be read from any thread without requiring synchronization primitives. The static factory method fromValue is also thread-safe.

## API Surface
The public contract is designed for serialization and deserialization of the notification type over the network.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for network serialization. |
| fromValue(int value) | static AssetEditorPopupNotificationType | O(1) | Deserializes an integer from the network into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the construction and parsing of packets that convey UI notifications for the Asset Editor. The server uses getValue to serialize the type, and the client uses fromValue to deserialize it.

```java
// Server-side: Preparing a packet for transmission
int networkValue = AssetEditorPopupNotificationType.Success.getValue();
// networkValue (1) is written to the packet buffer

// Client-side: Parsing an incoming packet
int receivedValue = packetBuffer.readVarInt();
AssetEditorPopupNotificationType type = AssetEditorPopupNotificationType.fromValue(receivedValue);
// The client UI system can now use 'type' to display the correct popup
```

### Anti-Patterns (Do NOT do this)
- **Manual Ordinal Comparison:** Do not rely on the `ordinal()` method for serialization. The integer values are explicitly defined and should be accessed via `getValue()`. Relying on ordinality is brittle and will break if the enum declaration order changes.
- **Ignoring Exceptions:** The `fromValue` method can throw a `ProtocolException`. Failure to handle this exception in the network layer can lead to client crashes or a desynchronized state if a malformed or outdated packet is received. Always wrap calls to `fromValue` in a try-catch block during packet parsing.

## Data Pipeline
AssetEditorPopupNotificationType is not a processing stage itself, but rather the data payload that flows through the notification pipeline.

> Flow:
> Server Asset Logic -> **AssetEditorPopupNotificationType.Error** -> Packet Serializer (`getValue`) -> Network Stream -> Packet Deserializer (`fromValue`) -> Client Event Bus -> Asset Editor UI System -> Render Error Popup

