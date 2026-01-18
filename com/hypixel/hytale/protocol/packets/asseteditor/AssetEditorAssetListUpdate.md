---
description: Architectural reference for AssetEditorAssetListUpdate
---

# AssetEditorAssetListUpdate

**Package:** com.hypixel.hytale.protocol.packets.asseteditor
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AssetEditorAssetListUpdate implements Packet {
```

## Architecture & Concepts
The AssetEditorAssetListUpdate class is a network packet definition within the Hytale Protocol Layer. It serves as a specialized Data Transfer Object (DTO) designed to communicate incremental changes to an asset collection between the server and a client using the in-game asset editor.

This packet is not a complete snapshot of the asset list. Instead, it represents a *delta* or a *diff*, containing only the files that have been added or removed since the last synchronization. This design pattern is critical for minimizing network bandwidth, especially when managing large asset packs where only a few files change at a time.

The binary serialization format is highly optimized for performance. It employs a fixed-size header block containing a bitfield to track nullable fields, followed by integer offsets pointing to the location of variable-length data (such as strings and arrays) in a subsequent data block. This structure avoids the overhead of markers or terminators for every field, allowing for fast, direct memory access during deserialization.

## Lifecycle & Ownership
- **Creation:**
    - **Receiving End:** Instantiated exclusively by the protocol's deserialization pipeline when an incoming network buffer is identified with Packet ID 320. The static factory method *deserialize* is the designated entry point.
    - **Sending End:** Instantiated by higher-level game logic, typically an asset management service, to encapsulate a set of file changes that need to be transmitted.
- **Scope:** This object is ephemeral and has a very short lifecycle. It is designed to exist only long enough to be processed by a network handler or a game logic update cycle. It does not persist between ticks or network events.
- **Destruction:** The object becomes eligible for garbage collection immediately after its data has been consumed by the receiving system (e.g., after the local asset cache has been updated). There is no manual memory management required.

## Internal State & Concurrency
- **State:** The object is a mutable container for asset change data. Its public fields can be modified after construction. It holds no cached data or derived state; it is a pure data structure.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single-threaded context, such as a Netty event loop or a main game thread. Accessing or modifying an instance from multiple threads concurrently will lead to race conditions and unpredictable behavior. Any cross-thread usage must be managed with external synchronization mechanisms.

## API Surface
The primary contract of this class revolves around its static serialization and validation methods, not typical object-oriented interaction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | AssetEditorAssetListUpdate | O(N) | **[Factory]** Constructs a new instance by reading binary data from a Netty ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Encodes the object's state into a binary representation and writes it to the provided ByteBuf. |
| validateStructure(buf, offset) | ValidationResult | O(N) | Performs a read-only check of the binary data in a buffer to ensure it conforms to the packet structure without full deserialization. Critical for security and stability. |
| computeSize() | int | O(N) | Calculates the exact number of bytes the object will occupy when serialized. Used for pre-allocating buffers. |

*N represents the total number of entries in the additions and deletions arrays.*

## Integration Patterns

### Standard Usage
This packet is handled by the network protocol layer. Application code typically creates an instance, populates it, and hands it off to a network service for dispatch.

```java
// Example: Sending an update from the server
AssetEditorFileEntry newTexture = new AssetEditorFileEntry("textures/swords/diamond.png", hash1);
AssetEditorFileEntry deletedModel = new AssetEditorFileEntry("models/mobs/creeper.hymodel", hash2);

AssetEditorAssetListUpdate updatePacket = new AssetEditorAssetListUpdate(
    "core_assets",
    new AssetEditorFileEntry[]{newTexture},
    new AssetEditorFileEntry[]{deletedModel}
);

// The network manager will internally call updatePacket.serialize(buffer)
networkManager.sendPacket(player, updatePacket);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation After Dispatch:** Do not modify the packet's fields after passing it to a network manager. The serialization may occur on a separate network thread, leading to a race condition where an incomplete or corrupted object state is sent.
- **Bypassing Validation:** On the receiving end, failing to call *validateStructure* on data from an untrusted source before attempting to *deserialize* can expose the application to denial-of-service attacks. A malformed packet could specify an enormous array length, leading to an OutOfMemoryError.
- **Manual Deserialization:** Do not attempt to read the fields from a ByteBuf manually. The binary layout is complex and relies on the precise logic within the static *deserialize* method.

## Data Pipeline
The AssetEditorAssetListUpdate packet is a message that flows through the network stack to synchronize state.

> **Sending Flow:**
> Asset Service (Detects Change) -> **AssetEditorAssetListUpdate (Instance Created)** -> Protocol Layer (serialize is called) -> Netty Channel -> Remote Peer

> **Receiving Flow:**
> Remote Peer -> Netty Channel -> Protocol Layer (validateStructure and deserialize are called) -> **AssetEditorAssetListUpdate (Instance Hydrated)** -> Packet Handler -> Asset Cache (State Updated)

