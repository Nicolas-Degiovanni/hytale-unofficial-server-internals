---
description: Architectural reference for CosmeticsModule
---

# CosmeticsModule

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Singleton

## Definition
```java
// Signature
public class CosmeticsModule extends JavaPlugin {
```

## Architecture & Concepts
The CosmeticsModule is a core server plugin responsible for managing all aspects of player appearance and customization. It functions as the central authority for creating, validating, and processing player cosmetic data, known as a PlayerSkin.

As a JavaPlugin, it is loaded and managed by the server's plugin lifecycle system. Its primary responsibilities include:

*   **Registry Management:** It owns and manages a single instance of CosmeticRegistry, which acts as an in-memory database of all available cosmetic parts (e.g., faces, hair, clothing) loaded from the game's asset packs.
*   **Skin Validation:** It provides a robust validation pipeline to ensure that any given PlayerSkin object is composed of valid, compatible, and existing cosmetic parts. This is critical for maintaining server stability and preventing malformed player appearance data from entering the game world.
*   **Model Instantiation:** It serves as a factory for converting abstract PlayerSkin data structures into renderable Model objects. This process bridges the gap between a player's cosmetic choices and their actual in-game representation.
*   **System Integration:** The module integrates with several other core systems. It hooks into the AssetModule to populate its registry, registers commands like EmoteCommand with the CommandRegistry, and listens for engine-level events like LoadAssetEvent to perform asset integrity checks during server startup.

## Lifecycle & Ownership
- **Creation:** The CosmeticsModule is instantiated once by the server's plugin loader during the initial bootstrap sequence. The static singleton instance is set within the constructor, ensuring it is available shortly after creation.
- **Scope:** The instance persists for the entire runtime of the server. It is a foundational service that other systems rely on.
- **Destruction:** The object is dereferenced and eligible for garbage collection during the server shutdown process, as the plugin system unloads all active plugins.

## Internal State & Concurrency
- **State:** The module's primary state is held within the CosmeticRegistry instance. This registry is populated once during the plugin's setup phase and is treated as a read-only cache thereafter. The state is therefore considered mutable during initialization but effectively immutable during normal operation.
- **Thread Safety:** The class is not explicitly thread-safe and does not employ internal locking. Initialization via the setup method is guaranteed to be safe as it occurs on the main server thread during startup. Subsequent read operations like validateSkin or createModel are safe for concurrent access under the assumption that the underlying CosmeticRegistry is not modified after the setup phase is complete.

**Warning:** Any external modification to the CosmeticRegistry after server startup will break thread-safety guarantees and lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static CosmeticsModule | O(1) | Retrieves the global singleton instance. |
| getRegistry() | CosmeticRegistry | O(1) | Returns the underlying registry of all cosmetic parts. |
| createModel(PlayerSkin, float) | Model | O(P) | Validates a skin and creates a scaled 3D model. P is the number of parts on the skin. Returns null on validation failure. |
| validateSkin(PlayerSkin) | void | O(P) | Verifies that all parts of a skin are valid. Throws InvalidSkinException on failure. |
| generateRandomSkin(Random) | PlayerSkin | O(P) | Constructs a new, valid PlayerSkin with randomly selected parts. P is the number of cosmetic slots. |

## Integration Patterns

### Standard Usage
The module should always be accessed via its static get method. It is commonly used to validate and create player models from network data or for NPC generation.

```java
// How a developer should normally use this
CosmeticsModule cosmetics = CosmeticsModule.get();
PlayerSkin skin = getSkinFromPlayer(); // Assume this exists

try {
    // Explicit validation is often a good practice before expensive operations
    cosmetics.validateSkin(skin);
    Model playerModel = cosmetics.createModel(skin, 1.0f);

    if (playerModel != null) {
        // Add model to the world
    }
} catch (CosmeticsModule.InvalidSkinException e) {
    // Handle the invalid skin, perhaps by kicking the player or logging an error
    getLogger().warning("Player provided an invalid skin!");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CosmeticsModule()`. The plugin is instantiated by the server's core. Manual creation will result in a non-functional, detached object and will break the singleton pattern.
- **Premature Access:** Do not call `CosmeticsModule.get()` from another plugin's constructor. The singleton instance is only guaranteed to be available after the plugin's `setup()` method has been called by the server.
- **State Mutation:** Do not attempt to modify the returned CosmeticRegistry after server initialization. The registry is not designed for runtime modification and doing so can cause validation failures and race conditions.

## Data Pipeline
The CosmeticsModule primarily processes data in two distinct flows: validating an existing skin to create a model, and validating game assets during startup.

**Model Creation Flow**
This is the most common runtime data flow, typically triggered when a player joins the server or changes their appearance.

> Flow:
> PlayerSkin object -> `CosmeticsModule.createModel` -> `validateSkin` -> CosmeticRegistry lookup -> `Model.createScaledModel` -> Renderable Model

**Asset Validation Flow**
This flow only occurs during server startup if the `VALIDATE_ASSETS` option is enabled. It is a self-contained integrity check.

> Flow:
> LoadAssetEvent -> `validateGeneratedSkin` -> `generateRandomSkin` -> `validateSkin` -> LoadAssetEvent.failed() (on error)

