---
description: Architectural reference for PrefabEditorCreationSettings
---

# PrefabEditorCreationSettings

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Transient

## Definition
```java
// Signature
public class PrefabEditorCreationSettings
   implements PrefabEditorCreationContext,
   JsonAssetWithMap<String, DefaultAssetMap<String, PrefabEditorCreationSettings>> {
```

## Architecture & Concepts
The PrefabEditorCreationSettings class is a configuration object that encapsulates all parameters required for a prefab loading or creation session within the Hytale builder tools. It serves as the primary data structure for defining *how* and *what* to load into the prefab editor's dedicated world space.

Architecturally, this class acts as a bridge between user input (either from a command like `/editprefab` or a saved JSON preset) and the server's core prefab processing systems. By implementing the `PrefabEditorCreationContext` interface, it presents a stable contract to the loading machinery, abstracting away the details of its own creation and configuration.

Its dual nature is critical:
1.  **As a Data Transfer Object (DTO):** It collects and holds numerous settings, such as file paths, placement rules (alignment, stacking axis), world generation parameters, and entity loading flags.
2.  **As a Serializable Asset:** By implementing `JsonAssetWithMap` and defining a static `CODEC`, instances can be serialized to and from JSON files. This allows developers and builders to create reusable presets for complex prefab arrangements, which can be loaded by name.

The class distinguishes between `unprocessedPrefabPaths` (raw string inputs) and `prefabPaths` (resolved, validated `java.nio.file.Path` objects). This separation is intentional, deferring file system interaction and validation to a distinct finalization step.

## Lifecycle & Ownership
-   **Creation:** An instance is created in one of two ways:
    1.  **Deserialization:** The `AssetStore` instantiates the class via the static `CODEC` when a preset is loaded from a JSON file using the static `load` method.
    2.  **Programmatic Construction:** A command handler, such as `PrefabEditLoadCommand`, instantiates the class and populates its fields based on arguments provided by a player.

-   **Scope:** The object is operation-scoped. Its lifetime is tied to a single prefab loading command execution. It is created, finalized, used to configure the loading process, and then becomes eligible for garbage collection. It should not be cached or reused across different operations.

-   **Destruction:** Managed by the Java garbage collector once all references are released after the prefab loading operation is complete.

## Internal State & Concurrency
-   **State:** The class is highly mutable and stateful. Its initial state is considered "unprocessed". A critical state transition occurs during the `finishProcessing` method call. This method resolves the raw string paths, performs security validation, populates the transient `prefabPaths` list, and captures the player context.

    Fields marked as `transient` (e.g., `player`, `playerRef`, `prefabPaths`) represent runtime-only state that is not persisted to JSON. This cleanly separates the persistent configuration from the immediate operational context.

-   **Thread Safety:** This class is **not thread-safe**. It is designed to be created, configured, and used exclusively on the main server thread. All mutations and method calls must be synchronized externally if multi-threaded access is required.

    While the static `load` and `save` methods return a `CompletableFuture` and perform I/O on a background thread, the resulting `PrefabEditorCreationSettings` object is intended to be consumed by the main thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| finishProcessing(Player, PlayerRef, boolean) | PrefabEditorCreationContext | I/O Bound | Finalizes the settings. Resolves paths, performs security checks, and validates file existence. Returns null on failure. **This must be called before use.** |
| load(String) | CompletableFuture | I/O Bound | Asynchronously loads and deserializes a settings preset from the AssetStore. |
| save(String, settings) | CompletableFuture | I/O Bound | Asynchronously serializes and saves the settings instance to a named preset in the AssetStore. |
| getAssetStore() | AssetStore | O(1) | Static accessor for the underlying asset management system for this class. |
| getPrefabPaths() | List<Path> | O(1) | Returns the list of fully resolved, validated prefab file paths. Only valid after `finishProcessing`. |
| getStackingAxis() | PrefabStackingAxis | O(1) | Returns the axis along which multiple prefabs should be arranged. |

## Integration Patterns

### Standard Usage
The primary pattern involves loading a named preset, finalizing it with the current player's context, and then passing it to a system that consumes a `PrefabEditorCreationContext`.

```java
// How a developer should normally use this
Player editor = ...;
PlayerRef editorRef = ...;

PrefabEditorCreationSettings.load("complex_dungeon_layout").thenAccept(settings -> {
    if (settings == null) {
        // Handle preset not found
        return;
    }

    // Finalize the settings with the current player context
    PrefabEditorCreationContext context = settings.finishProcessing(editor, editorRef, false);

    if (context != null) {
        // Pass the validated context to the prefab loading system
        PrefabLoader.loadFromContext(context);
    } else {
        // finishProcessing failed; error messages were already sent to the player
    }
});
```

### Anti-Patterns (Do NOT do this)
-   **Skipping Finalization:** Do not use an instance of this class before calling `finishProcessing`. The internal list of resolved `prefabPaths` will be empty, and the player context will be null, leading to `NullPointerException` or silent failure downstream.

-   **Reusing Instances:** Do not cache and reuse a `PrefabEditorCreationSettings` object for a new operation. The internal state, especially the player context and resolved paths, is specific to the single operation during which `finishProcessing` was called.

-   **Ignoring Security Context:** The `finishProcessing` method performs critical security checks, such as preventing path traversal attacks (`../`) when running on a server. Bypassing this method and resolving paths manually is dangerous.

## Data Pipeline
The class acts as a configuration hub in the prefab loading data flow. It translates high-level requests into a concrete, validated execution plan.

> **Flow (Preset):**
> JSON File on Disk -> `AssetStore` -> `CODEC` Deserialization -> **PrefabEditorCreationSettings** (Unprocessed) -> `finishProcessing()` -> **PrefabEditorCreationSettings** (Processed Context) -> Prefab Loading System -> World Modification

> **Flow (Command):**
> Player Command Input -> Command Parser -> **PrefabEditorCreationSettings** (Unprocessed) -> `finishProcessing()` -> **PrefabEditorCreationSettings** (Processed Context) -> Prefab Loading System -> World Modification

