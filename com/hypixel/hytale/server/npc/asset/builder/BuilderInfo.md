---
description: Architectural reference for BuilderInfo
---

# BuilderInfo

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BuilderInfo {
```

## Architecture & Concepts
The BuilderInfo class is a stateful metadata container, acting as a tracking record for a single, buildable asset within the server's asset management system. It does not perform any loading or building itself; instead, it encapsulates the state of an asset as it moves through the various stages of its lifecycle: discovery, validation, loading, and removal.

Architecturally, this class serves as the atomic unit of work within a larger collection managed by a system like an AssetManager or ConfigurationLoader. By holding a reference to a generic Builder, a file Path, and a unique keyName, it decouples the management logic from the specific type of asset being processed. The core of its design is the internal state machine, represented by the State enum, which dictates whether an asset needs to be validated, reloaded, or can be considered active.

This object is fundamental to the hot-reloading and dynamic configuration capabilities of the server, allowing the system to efficiently track which assets have changed and require reprocessing without a full server restart.

### Lifecycle & Ownership
- **Creation:** An instance of BuilderInfo is created by a managing service when a potential asset (e.g., a JSON file for an NPC) is discovered on the filesystem. The manager is responsible for providing the initial context, such as its index, key, and the associated Builder.
- **Scope:** The lifetime of a BuilderInfo object is strictly tied to the asset it represents. It persists as long as the managing service keeps a reference to it, typically for the duration that the asset is considered active or known to the server.
- **Destruction:** The object is marked for garbage collection when the managing service releases its reference. This typically occurs when an asset file is deleted from the disk and the system performs a cleanup cycle. The setRemoved method explicitly transitions the state to signal this intent.

## Internal State & Concurrency
- **State:** BuilderInfo is a highly mutable object. Its primary purpose is to mutate the internal status field to reflect the current state of the associated asset. It is designed to be a lightweight state tracker, not an immutable data record.
- **Thread Safety:** This class is **not thread-safe**. All methods that modify the internal status are unsynchronized. Concurrent modification of a BuilderInfo instance from multiple threads will lead to race conditions and unpredictable behavior. All access and mutation must be externally synchronized, typically by being confined to a single asset-processing thread or through explicit locking within the managing service.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setValidated(boolean) | boolean | O(1) | Finalizes the validation state. Transitions status to VALID or INVALID. |
| setNeedsValidation() | void | O(1) | Resets the status to NEEDS_VALIDATION. Used when a dependency changes. |
| setNeedsReload() | void | O(1) | Marks the asset as requiring a full reload from disk. This is a higher priority state than validation. |
| setRemoved() | void | O(1) | Marks the asset as removed. This is a terminal state. |
| canBeValidated() | boolean | O(1) | Checks if the current state allows a validation attempt. Returns false if removed or needing a reload. |
| isValid() | boolean | O(1) | Returns true only if the status is explicitly VALID. |

## Integration Patterns

### Standard Usage
BuilderInfo is intended to be managed within a collection by a higher-level service. The service iterates over its collection of BuilderInfo objects, inspects their state, and orchestrates the necessary actions like validation or reloading.

```java
// A manager class processing a list of discovered assets
for (BuilderInfo info : allAssetInfos) {
    if (info.needsValidation()) {
        // The builder is retrieved from the info object to perform the work
        Builder<?> assetBuilder = info.getBuilder();
        boolean success = assetBuilder.validate();
        info.setValidated(success);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mismanagement:** Do not hold a reference to a BuilderInfo object and assume its state is static. Always re-check its status (e.g., isValid) before using the associated asset, as a file-watcher or other system may have changed its state.
- **Concurrent Modification:** Never modify a BuilderInfo instance from multiple threads without external locking. This will corrupt the state machine.
- **Orphaned Instances:** Do not create BuilderInfo instances without registering them in a managing collection. They serve no purpose outside of this context and will not be processed by the engine.

## Data Pipeline
BuilderInfo acts as a state-tracking node within the server's asset processing pipeline. It does not transform data itself but records the outcome of operations performed by other systems.

> Flow:
> Filesystem Watcher (File Created/Modified) -> AssetManager creates/updates **BuilderInfo (NEEDS_VALIDATION)** -> Validation Service processes the Builder -> **BuilderInfo (VALID / INVALID)** -> Game uses the built asset.

