---
description: Architectural reference for ModifiedAsset
---

# ModifiedAsset

**Package:** com.hypixel.hytale.builtin.asseteditor.data
**Type:** Data Model (Transient)

## Definition
```java
// Signature
public class ModifiedAsset {
```

## Architecture & Concepts
The **ModifiedAsset** class is a fundamental data structure within the Asset Editor subsystem. It serves as an in-memory representation of a pending change to a single asset, such as a texture, model, or configuration file. This class is not merely a passive data container; it encapsulates the complete context of a modification, including its current state (new, changed, deleted), its location, and a full audit trail of the change.

Its primary architectural role is to act as a standardized, serializable unit of work. The static **CODEC** field is central to this design, providing a robust mechanism for serializing the object's state to disk or for network transmission. This allows the system to persist the state of uncommitted changes and to synchronize asset modifications between a server and multiple connected editor clients.

The class design explicitly handles legacy data formats by mapping older boolean fields like **IsNew** and **IsDeleted** to the modern **AssetState** enum during deserialization, ensuring backward compatibility.

## Lifecycle & Ownership
- **Creation:** An instance of **ModifiedAsset** is created whenever a user action modifies an asset within the editor. This can be triggered by creating a new file, saving an existing one, renaming, or deleting. Instances are also created by the **CODEC** during deserialization when loading project state from disk.
- **Scope:** The object's lifetime is intentionally short. It exists only as long as a change is considered "pending" or "dirty". It is owned by a higher-level manager, such as an **AssetEditorService**, which orchestrates the modification workflow.
- **Destruction:** The object is eligible for garbage collection once its state has been processed. This typically occurs after it has been serialized to a persistent format or converted into a network packet (via **toAssetInfoPacket**) and dispatched.

## Internal State & Concurrency
- **State:** The internal state is highly mutable. All data-holding fields are public, allowing direct modification by the owning system. This design prioritizes performance and simplicity within a single-threaded context. The object's state represents a snapshot of a change at a specific point in time.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locks or synchronization mechanisms. All access and mutation must be externally synchronized, typically by confining its use to a single, dedicated thread (e.g., the main application thread). Unmanaged concurrent access will lead to data corruption and unpredictable behavior.

## API Surface
The public API is minimal, focusing on state mutation and data transformation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| markEditedBy(EditorClient) | void | O(1) | Populates the audit trail fields (timestamp, username, UUID) from the provided **EditorClient**. This method should be called immediately after a modification. |
| toAssetInfoPacket(String) | AssetInfo | O(1) | Transforms the internal state into an **AssetInfo** network packet. This is the primary mechanism for preparing the modification data for client-server communication. |

## Integration Patterns

### Standard Usage
The intended use involves a controlling service creating an instance, populating its state, and then using it for serialization or network dispatch.

```java
// An owning service creates and populates the object after a user action
ModifiedAsset pendingChange = new ModifiedAsset();
pendingChange.path = Path.of("models/items/sword.json");
pendingChange.dataFile = Path.of("/tmp/asset-editor/temp-123.json");
pendingChange.state = AssetState.CHANGED;

// The audit trail is stamped
pendingChange.markEditedBy(editorClient);

// The change is prepared for network dispatch
AssetInfo packet = pendingChange.toAssetInfoPacket("core");
networkManager.send(packet);
```

### Anti-Patterns (Do NOT do this)
- **Long-Lived References:** Do not retain instances of **ModifiedAsset** after the corresponding change has been committed. They represent transient state and should be discarded to avoid memory leaks and stale data.
- **Cross-Thread Modification:** Never modify an instance from a different thread than the one that created it without implementing external locking. This will cause race conditions.
- **Manual Serialization:** Do not attempt to serialize this object using custom logic. The static **CODEC** is the *only* supported mechanism and is designed to handle versioning and data migration correctly. Bypassing it will break data compatibility.

## Data Pipeline
**ModifiedAsset** acts as a critical intermediate step in the flow of asset data from user input to network replication.

> Flow:
> User Action in Editor -> Owning Service creates **ModifiedAsset** -> **ModifiedAsset** is populated -> **toAssetInfoPacket()** -> AssetInfo Packet -> Network Layer -> Remote Editor Client

