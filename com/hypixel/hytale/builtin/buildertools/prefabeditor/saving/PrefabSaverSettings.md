---
description: Architectural reference for PrefabSaverSettings
---

# PrefabSaverSettings

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.saving
**Type:** Transient

## Definition
```java
// Signature
public class PrefabSaverSettings {
```

## Architecture & Concepts
PrefabSaverSettings is a **Parameter Object**, a design pattern used to encapsulate the configuration for a complex operation into a single, transferable object. Its primary role is to configure the behavior of the prefab serialization and saving process, governed by the PrefabSaver service.

This class decouples the user interface or command layer, which determines *how* a prefab should be saved, from the underlying saving mechanism that performs the work. By bundling numerous boolean flags into one object, it simplifies the method signatures of the saving services and provides a clear, self-documenting contract for the save operation's parameters. It is a plain data-holder with no internal logic.

## Lifecycle & Ownership
- **Creation:** An instance of PrefabSaverSettings is typically created on-demand by the Prefab Editor's UI logic in response to a user initiating a save action. The state of UI elements (like checkboxes) is used to populate the fields of the new instance.
- **Scope:** The object's lifetime is extremely short and is scoped to a single save operation. It is created, passed as an argument to the saving service, and then immediately becomes eligible for garbage collection once the operation completes.
- **Destruction:** The Java Garbage Collector is responsible for deallocating the object. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency
- **State:** The internal state is entirely **mutable** and consists of a collection of boolean flags. The object is designed to be instantiated and then configured via its public setters before being used. It holds no derived or cached data.
- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization mechanisms. It is intended to be created, populated, and consumed within a single thread, typically the main client thread.

**WARNING:** Sharing an instance of PrefabSaverSettings across multiple threads will lead to race conditions and unpredictable behavior. Do not store instances in static fields or pass them to asynchronous tasks without proper external synchronization.

## API Surface
The public API consists exclusively of standard getters and setters to configure the save operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setRelativize(boolean) | void | O(1) | Configures whether the prefab's coordinates should be saved relative to its anchor point. |
| setOverwriteExisting(boolean) | void | O(1) | If true, allows the save operation to overwrite an existing prefab file with the same name. |
| setBlocks(boolean) | void | O(1) | Determines if block data should be included in the saved prefab. |
| setEntities(boolean) | void | O(1) | Determines if entity data should be included in the saved prefab. |
| setKeepAnchors(boolean) | void | O(1) | Configures whether anchor entities should be preserved within the saved prefab data. |

## Integration Patterns

### Standard Usage
The intended use is to instantiate, configure, and pass the object to a saving service within the scope of a single method call.

```java
// A UI controller or command handler creates and configures the settings
PrefabSaverSettings settings = new PrefabSaverSettings();
settings.setRelativize(uiCheckboxRelativize.isChecked());
settings.setOverwriteExisting(true);
settings.setBlocks(true);
settings.setEntities(false);

// The configured object is passed to the responsible service
PrefabSaver saver = context.getService(PrefabSaver.class);
saver.savePrefab(selectedPrefab, targetPath, settings);
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not cache and re-use a PrefabSaverSettings instance for multiple, distinct save operations. This can lead to stale configuration being applied to a new save. Always create a fresh instance.
- **Shared State:** Do not store an instance in a static field or other long-lived scope. It is a transient data container, not a persistent configuration service.

## Data Pipeline
PrefabSaverSettings does not process data itself; instead, it acts as a configuration manifest that directs the flow and transformation of data within the PrefabSaver.

> Flow:
> User Input (UI) -> **PrefabSaverSettings** (Instantiation & Population) -> PrefabSaver Service -> Data Serialization (Guided by Settings) -> File System Write

