---
description: Architectural reference for AssetEditorFileTree
---

# AssetEditorFileTree

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum AssetEditorFileTree {
```

## Architecture & Concepts
The AssetEditorFileTree enum serves as a strict type-safe discriminator for identifying the root namespace of a file being manipulated within the Asset Editor protocol. It is a critical component of the network serialization layer, translating between a human-readable concept (e.g., a server-specific asset) and its compact, integer-based wire format representation.

This enum enforces a clear contract for any network packet related to file tree operations. By providing explicit Server and Common roots, it prevents ambiguity in asset paths and ensures that client and server logic operate on the correct virtual filesystem. The primary design function is to provide robust serialization and deserialization, with built-in validation to reject malformed or unsupported data streams immediately at the protocol boundary.

### Lifecycle & Ownership
- **Creation:** Instances are created and initialized by the Java Virtual Machine during class loading. As a static enum, its constants (Server, Common) are instantiated once and exist as singletons.
- **Scope:** Application-wide. The enum and its instances persist for the entire lifetime of the application.
- **Destruction:** The instances are reclaimed by the JVM during application shutdown. There is no manual memory management or destruction process.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a final primitive integer, and its state cannot be modified after construction. The VALUES array is also static and final.
- **Thread Safety:** Inherently thread-safe. Due to its immutability, AssetEditorFileTree can be safely accessed, read, and passed between any number of concurrent threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Serializes the enum to its integer wire format value. |
| fromValue(int value) | AssetEditorFileTree | O(1) | Deserializes an integer from the wire format into an enum instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is intended to be used exclusively by packet serialization and deserialization logic. The fromValue method is the designated factory for converting network data into a safe object, while getValue is used to prepare an object for network transmission.

```java
// Deserializing an incoming value from a network buffer
int fileTreeId = buffer.readVarInt();
AssetEditorFileTree targetTree = AssetEditorFileTree.fromValue(fileTreeId);

// Use the resulting enum in game logic
if (targetTree == AssetEditorFileTree.Server) {
    // Process a server-specific asset request
}
```

### Anti-Patterns (Do NOT do this)
- **Integer Comparison:** Do not use the raw integer value for logical comparisons. This practice is brittle, bypasses the type system, and couples application logic directly to the network wire format.

    ```java
    // BAD: Couples logic to the wire format
    if (tree.getValue() == 0) { ... }

    // GOOD: Uses type-safe object comparison
    if (tree == AssetEditorFileTree.Server) { ... }
    ```
- **Unhandled Exceptions:** Failure to handle the ProtocolException thrown by fromValue will result in an unhandled exception in the network processing thread, potentially disconnecting the client. Always wrap calls in a try-catch block during packet decoding.

## Data Pipeline
The AssetEditorFileTree acts as a validation and translation gateway during the decoding phase of the network protocol.

> Flow:
> Network Byte Stream -> Protocol Decoder (reads integer) -> **AssetEditorFileTree.fromValue()** -> Populated Packet Object -> Asset Editor Service Logic

