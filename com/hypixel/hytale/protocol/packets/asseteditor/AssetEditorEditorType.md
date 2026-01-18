---
description: Architectural reference for AssetEditorEditorType
---

# AssetEditorEditorType

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Utility

## Definition
```java
// Signature
public enum AssetEditorEditorType {
   None(0),
   Text(1),
   JsonSource(2),
   JsonConfig(3),
   Model(4),
   Texture(5),
   Animation(6);
```

## Architecture & Concepts

AssetEditorEditorType is a type-safe enumeration that serves as a protocol-level discriminator for different asset editor UIs. Its primary function is to provide a canonical, integer-based representation for the various editor types that can be invoked remotely between the server and the game client.

This enum acts as a contract within the asset editing data stream. By serializing to a compact integer, it ensures minimal network overhead. The server can command the client to open a specific editor for an asset (e.g., "open asset *player.model* with the *Model* editor") without needing any knowledge of the client's UI implementation classes. This decouples the server's game logic from the client's presentation layer.

The static `fromValue` method is the designated deserialization gateway, providing robust bounds-checking to protect the client from malformed or malicious packets.

## Lifecycle & Ownership
- **Creation:** All enum constants are instantiated once by the JVM during class loading. They are compile-time constants and exist as static singletons for the entire application lifetime.
- **Scope:** Application-wide. A single instance of each constant, such as AssetEditorEditorType.Model, is shared across all threads and components.
- **Destruction:** The enum constants are reclaimed by the JVM only when the application shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a final, private integer value that is set at compile time and can never be modified. The static VALUES array is also final.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, AssetEditorEditorType can be safely read, passed, and referenced from any thread without requiring locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the stable, integer representation of the enum constant, intended for network serialization. |
| fromValue(int value) | AssetEditorEditorType | O(1) | **[Factory]** Deserializes an integer from a network stream into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage

This enum is used during the serialization and deserialization of packets that trigger the asset editor.

**Encoding a Packet (Server-side):**
```java
// When sending a command to the client
OpenEditorPacket packet = new OpenEditorPacket();
packet.setEditorType(AssetEditorEditorType.Model.getValue());
// ... serialize and send packet
```

**Decoding a Packet (Client-side):**
```java
// When a packet arrives from the server
int rawTypeValue = receivedPacket.readInt();
try {
    AssetEditorEditorType editorType = AssetEditorEditorType.fromValue(rawTypeValue);
    // Use the type to open the correct UI
    EditorFactory.openEditorFor(assetId, editorType);
} catch (ProtocolException e) {
    // Handle protocol error, typically by logging and disconnecting the client
    connection.disconnect("Invalid editor type received.");
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinal for Serialization:** Never use the built-in `ordinal()` method for serialization. If the order of enum constants is changed in the future, it will break protocol compatibility. This enum correctly uses a dedicated `value` field, which is the required pattern.
- **Ignoring ProtocolException:** The `fromValue` method can throw a ProtocolException. This is a critical error indicating a protocol mismatch or data corruption. Swallowing this exception can lead to undefined client behavior. The connection should be terminated.
- **Extending the Enum:** Enums in Java are final and cannot be extended. Custom logic should be implemented in services that consume this enum, not by attempting to modify it.

## Data Pipeline

AssetEditorEditorType is not a processing component but rather a data value that flows through the network pipeline. It dictates routing and behavior at the destination.

> Flow:
> Server Command -> `OpenEditorPacket` (contains integer from `getValue()`) -> Network Serialization -> **Wire Protocol (integer)** -> Network Deserialization -> `fromValue(integer)` -> Client-side Editor Factory -> UI Instantiation (e.g., ModelEditor)

