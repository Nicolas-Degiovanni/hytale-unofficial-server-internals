---
description: Architectural reference for CannedWarpNames
---

# CannedWarpNames

**Package:** com.hypixel.hytale.builtin.adventure.teleporter.util
**Type:** Utility

## Definition
```java
// Signature
public final class CannedWarpNames {
```

## Architecture & Concepts
CannedWarpNames is a stateless, server-side utility class responsible for generating deterministic, non-custom names for teleporters. It acts as a critical bridge between the entity-component system and global server modules, ensuring that newly activated teleporters receive a unique, procedurally generated name that does not conflict with existing warps in the same world.

Its core function is to provide a collision-free naming scheme by:
1.  Querying the global **TeleportPlugin** for a complete list of existing warp names within a specific world.
2.  Referencing a **WordList** asset, specified in the teleporter's own **Teleporter** component, as the source for potential names.
3.  Using a deterministic seed, derived from the teleporter block's **BlockStateInfo**, to select a name from the WordList. This ensures that the same teleporter block will consistently generate the same name, unless that name is already taken.
4.  Translating the chosen name key via the **I18nModule** to provide a localized name to the client.

This class centralizes the logic for procedural name generation, decoupling the Teleporter component from direct knowledge of global warp registries or localization systems.

## Lifecycle & Ownership
-   **Creation:** As a final class with a private constructor containing only static methods, CannedWarpNames is never instantiated. It is loaded into the JVM by the class loader when its methods are first referenced.
-   **Scope:** The class and its static methods exist for the entire lifetime of the server process.
-   **Destruction:** The class is unloaded from memory when the server's JVM shuts down.

## Internal State & Concurrency
-   **State:** This class is entirely **stateless**. It holds no member variables and does not cache any data between calls. Each method invocation computes its result based on the provided arguments and the current state of external systems like TeleportPlugin and I18nModule.

-   **Thread Safety:** The class itself has no internal state to protect, making its methods appear safe. However, its safety is **contingent on the thread safety of the modules it accesses**.
    -   **CRITICAL WARNING:** The method getWarpNamesInWorld iterates over the collection returned by `TeleportPlugin.get().getWarps()`. If another thread modifies this global warp collection while an invocation is in progress, a ConcurrentModificationException may be thrown. Any system calling CannedWarpNames must ensure that no concurrent writes to the world's warp list are occurring.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateCannedWarpName(blockRef, language) | String | O(N) | Generates a final, localized warp name. N is the number of warps in the world. Returns null if a name cannot be generated. |
| generateCannedWarpNameKey(blockRef, language) | String | O(N) | Generates the raw, non-localized translation key for a warp name. N is the number of warps in the world. Returns null if a key cannot be generated. |

## Integration Patterns

### Standard Usage
This utility should be called by server-side logic when a teleporter needs a default name, typically upon its initial creation or activation. The caller must provide a reference to the teleporter's block entity.

```java
// A server-side system has a reference to a teleporter block
Ref<ChunkStore> teleporterBlockRef = ...;
String userLanguage = "en_us";

// Generate and assign the name
String warpNameKey = CannedWarpNames.generateCannedWarpNameKey(teleporterBlockRef, userLanguage);

if (warpNameKey != null) {
    // The system would now update the Teleporter component with this key
    // and set its isCustomName flag to false.
}
```

### Anti-Patterns (Do NOT do this)
-   **Long-Term Storage of Keys:** Do not generate a name key with generateCannedWarpNameKey and store it for later use. The name generation process is dynamic and depends on the set of *currently existing* warps. A key generated at time T1 may no longer be unique at time T2. A name should be generated and immediately assigned to the Teleporter component.
-   **Attempted Instantiation:** Do not attempt to create an instance of CannedWarpNames via `new` or reflection. It is a static utility class.
-   **Ignoring Null Return:** The methods can return null if a unique name cannot be found (e.g., if the WordList is exhausted or missing). Calling code must be robust and handle the null case gracefully.

## Data Pipeline
The flow of data for generating a name is a multi-stage process involving several core server systems.

> Flow:
> 1.  **Input:** A `Ref<ChunkStore>` pointing to a teleporter block and a language code.
> 2.  **Component Extraction:** The method accesses the `Store` to retrieve the `World`, `BlockStateInfo`, and `Teleporter` components associated with the block reference.
> 3.  **Global State Query:** It calls `TeleportPlugin` to get a set of all existing warp names in the current `World`.
> 4.  **Name Selection:** It queries the `WordList` asset manager using a key from the `Teleporter` component. It then uses a seeded `Random` number generator (seeded by `BlockStateInfo.getIndex()`) to deterministically pick a name that is not present in the set from the previous step.
> 5.  **Localization:** The chosen name's translation key is passed to the `I18nModule`.
> 6.  **Output:** The `I18nModule` returns the final, localized `String` representing the warp name.

