---
description: Architectural reference for AssetState
---

# AssetState

**Package:** com.hypixel.hytale.builtin.asseteditor.data
**Type:** Enum

## Definition
```java
// Signature
public enum AssetState {
```

## Architecture & Concepts
AssetState is a fundamental enumeration that represents the modification status of an asset within the Asset Editor's workspace. It functions as a simple, type-safe state machine descriptor, providing a clear and constrained vocabulary for tracking changes to assets like models, textures, or configuration files.

This enum is central to the editor's change-tracking and persistence mechanisms. Systems that manage asset versions, handle undo/redo operations, or serialize workspace changes to disk rely on AssetState to determine the appropriate action for a given asset. It decouples the *what* (the asset) from its *status* (changed, new, or deleted), allowing for clean, state-driven logic.

## Lifecycle & Ownership
- **Creation:** As an enumeration, its instances (CHANGED, NEW, DELETED) are created and initialized by the Java Virtual Machine's ClassLoader when the AssetState class is first loaded. This process is automatic and occurs before any application code can reference the type.
- **Scope:** The instances are static, final, and exist for the entire lifetime of the application. They are effectively global singletons managed by the JVM.
- **Destruction:** The instances are garbage collected only when the ClassLoader that loaded them is itself unloaded, which typically only happens upon application shutdown.

## Internal State & Concurrency
- **State:** Enum instances are inherently immutable. Their internal state, defined at compile time, cannot be altered at runtime.
- **Thread Safety:** AssetState is unconditionally thread-safe. Its instances are constants and can be safely accessed and compared from any thread without synchronization. This is a guaranteed property of the Java language specification for enums.

## API Surface
The primary API consists of the predefined constant values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CHANGED | AssetState | O(1) | Represents an existing asset that has been modified since the last saved state. |
| NEW | AssetState | O(1) | Represents an asset that has been newly created in the workspace and does not exist in the base project. |
| DELETED | AssetState | O(1) | Represents an asset that has been marked for deletion from the workspace. |

## Integration Patterns

### Standard Usage
AssetState is intended to be used as a data field within asset metadata objects or as a parameter in event payloads. Its primary consumption pattern is through `switch` statements to branch logic based on an asset's status.

```java
// How a developer should normally use this
void processAsset(AssetMetadata metadata) {
    switch (metadata.getState()) {
        case NEW:
            // Logic for newly created assets
            createAssetOnDisk(metadata);
            break;
        case CHANGED:
            // Logic for modified assets
            updateAssetOnDisk(metadata);
            break;
        case DELETED:
            // Logic for deleted assets
            deleteAssetFromDisk(metadata);
            break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for logic or persistence. The order of enum constants can change between versions, leading to subtle and severe bugs. Always compare instances directly or use `name()`.
- **Extensibility:** Enums are final and cannot be extended. Do not attempt to create subclasses or add new states at runtime. If more states are needed, they must be added directly to the AssetState source file.

## Data Pipeline
AssetState does not process data itself; it is a metadata flag that travels *with* data through various editor systems. It provides context to the data it accompanies.

> Flow:
> User Action (e.g., edit texture) -> AssetMetadata Update (state set to **CHANGED**) -> Workspace Change Event -> Persistence Service -> Writes modified asset to disk

