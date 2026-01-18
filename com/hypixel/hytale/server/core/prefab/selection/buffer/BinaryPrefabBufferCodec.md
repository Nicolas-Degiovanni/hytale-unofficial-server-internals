---
description: Architectural reference for BinaryPrefabBufferCodec
---

# BinaryPrefabBufferCodec

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Singleton / Utility

## Definition
```java
// Signature
public class BinaryPrefabBufferCodec implements PrefabBufferCodec<ByteBuf> {
```

## Architecture & Concepts
The **BinaryPrefabBufferCodec** is a critical serialization component responsible for translating the in-memory **PrefabBuffer** object into a compact, versioned binary format, and vice-versa. It acts as the bridge between the abstract representation of a world structure (a prefab) and its persistent, transmissible form as a Netty **ByteBuf**.

This codec is the canonical implementation for saving prefabs to disk and is likely used in network transport for structure data. Its design prioritizes efficiency and forward-compatibility through several key mechanisms:

1.  **Versioning:** The binary format has an explicit version number. The `deserialize` method performs strict version checks, ensuring that the server does not attempt to load prefabs from an unsupported future version of the game.
2.  **Data Migration:** The system is tightly integrated with the **BlockMigration** asset system. When deserializing older prefabs, it can automatically remap legacy block names to their modern equivalents, allowing old content to function in newer versions of the engine.
3.  **Dictionary Encoding:** To minimize file size, the codec first writes a dictionary mapping integer IDs to string-based block and fluid names. The main data payload then references these compact integer IDs instead of repeating full string names for every block.
4.  **Bitmasking:** A bitmask field is used for each block entry to denote the presence of optional data, such as component state, fluid information, or placement chance. This avoids wasting space for default or absent values.

This class is the low-level engine for prefab persistence. Higher-level systems, such as world generation or building tools, rely on this codec to handle the underlying data format complexities.

## Lifecycle & Ownership
- **Creation:** A single static instance, **INSTANCE**, is created at class-loading time. This follows the Singleton pattern.
- **Scope:** The singleton instance is application-scoped and persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected by the JVM upon application shutdown. There is no explicit cleanup or destruction method.

## Internal State & Concurrency
- **State:** The **BinaryPrefabBufferCodec** is entirely stateless. It holds no instance fields that store data between method calls. All necessary information is passed as arguments to its `serialize` and `deserialize` methods.

- **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, multiple threads can safely invoke the `serialize` and `deserialize` methods concurrently on the static **INSTANCE**, provided they are operating on distinct **PrefabBuffer** and **ByteBuf** objects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(path, buffer) | PrefabBuffer | O(N) | Deserializes a binary **ByteBuf** into an in-memory **PrefabBuffer**. Throws **IllegalStateException** on version mismatch or data corruption. N is the number of blocks and entities. |
| serialize(prefabBuffer) | ByteBuf | O(N) | Serializes an in-memory **PrefabBuffer** into a new binary **ByteBuf**. Throws **IllegalStateException** if serialization of entity or block state fails. N is the number of blocks and entities. |

## Integration Patterns

### Standard Usage
The codec should always be accessed via its static **INSTANCE** field. It is typically used by a manager class responsible for loading prefab assets from disk.

```java
// Standard pattern for loading a prefab from a binary buffer
ByteBuf prefabData = ... // Loaded from file or network
try {
    PrefabBuffer prefab = BinaryPrefabBufferCodec.INSTANCE.deserialize(null, prefabData);
    // ... use the loaded prefab for world generation or placement
} catch (IllegalStateException e) {
    // Handle data corruption or version mismatch errors
    log.error("Failed to load prefab", e);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BinaryPrefabBufferCodec()`. The class is designed as a stateless singleton; always use the provided **INSTANCE**.
- **Ignoring Exceptions:** The `deserialize` method performs critical validation. Failure to catch its exceptions can lead to server instability if a corrupt or incompatible prefab file is loaded.
- **Manual Buffer Management:** The `serialize` method returns a newly allocated **ByteBuf**. The caller is responsible for its lifecycle, including releasing it after it has been written to disk or a network socket to prevent memory leaks.

## Data Pipeline
The codec serves as a bidirectional translator between the server's internal object model and a serialized byte stream.

**Serialization (Writing a Prefab)**
> Flow:
> **PrefabBuffer** (In-Memory Object) -> **BinaryPrefabBufferCodec**.serialize() -> **ByteBuf** (Binary Data) -> File System or Network

**Deserialization (Reading a Prefab)**
> Flow:
> File System or Network -> **ByteBuf** (Binary Data) -> **BinaryPrefabBufferCodec**.deserialize() -> **PrefabBuffer** (In-Memory Object)

