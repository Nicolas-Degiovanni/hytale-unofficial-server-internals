---
description: Architectural reference for StaminaModule
---

# StaminaModule

**Package:** com.hypixel.hytale.server.core.modules.entity.stamina
**Type:** Singleton

## Definition
```java
// Signature
public class StaminaModule extends JavaPlugin {
```

## Architecture & Concepts
The StaminaModule is a server-side plugin that introduces stamina-related mechanics into the core gameplay engine. It operates as an extension to the server's Entity-Component-System (ECS) framework, building upon the foundational EntityModule and EntityStatsModule.

Its primary architectural function is to register new systems, components, and configuration types that collectively model stamina consumption and regeneration for entities. By integrating directly with the ECS registries, it ensures that stamina logic is processed efficiently within the main server game loop.

This module is not a standalone service but rather a provider of behaviors. It injects the following primitives into the engine:
*   **ECS System:** The SprintStaminaEffectSystem contains the core logic for how stamina is affected by actions like sprinting.
*   **ECS Resource:** The SprintStaminaRegenDelay component is attached to entities to manage their specific stamina state.
*   **Configuration:** The StaminaGameplayConfig allows designers to tune stamina parameters through external asset files, decoupling game balance from engine code.

## Lifecycle & Ownership
- **Creation:** The StaminaModule is instantiated exclusively by the server's plugin loading mechanism during server bootstrap. Its existence and dependencies are declared in its static PluginManifest.
- **Scope:** The module is session-scoped. A single instance is created when the server starts and persists until the server shuts down.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection during the server's shutdown sequence when all plugins are unloaded.

## Internal State & Concurrency
- **State:** The StaminaModule instance itself is largely stateless. Its primary role is initialization and registration. It holds a reference to the registered ResourceType for SprintStaminaRegenDelay, but the actual stamina data for each entity is stored within the central EntityStore, not in this module.
- **Thread Safety:** This class is not designed for concurrent access from arbitrary threads. The setup method is invoked synchronously on the server's main thread during initialization. The static get method provides a thread-safe way to retrieve the singleton instance, but any subsequent interaction with the ECS registries it provides access to must conform to the server's threading model. The onGameplayConfigsLoaded event handler may be called from an asset loading thread and is designed to trigger a thread-safe cache invalidation.

## API Surface
The public contract of this module is minimal, as its primary purpose is to register behavior with the engine rather than to be called directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static StaminaModule | O(1) | Retrieves the global singleton instance of the module. Throws if called before plugin initialization. |
| getSprintRegenDelayResourceType() | ResourceType | O(1) | Returns the unique type handle for the SprintStaminaRegenDelay component. Used for ECS queries. |

## Integration Patterns

### Standard Usage
Direct interaction with the StaminaModule is uncommon. The standard pattern is to interact with the components and systems it registers. For example, another system might add the stamina component to an entity to enable stamina mechanics for it.

```java
// Example of another system enabling stamina for an entity
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

// Retrieve the resource type handle once
ResourceType<EntityStore, SprintStaminaRegenDelay> delayType = StaminaModule.get().getSprintRegenDelayResourceType();

// Add the component to a specific entity
EntityStore entityStore = ...;
entityStore.addResource(entityId, delayType);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new StaminaModule(). The server's plugin loader is responsible for its lifecycle. Attempting to create an instance manually will break the server's state and lead to critical failures.
- **Early Access:** Do not call StaminaModule.get() from a plugin constructor or before the plugin system has completed its loading phase. This will result in a NullPointerException as the static instance will not have been assigned yet.
- **Configuration Tampering:** Do not attempt to modify stamina configuration values at runtime by bypassing the asset system. The module relies on the LoadedAssetsEvent to correctly process configuration changes.

## Data Pipeline
The module participates in two primary data flows: configuration loading and gameplay logic processing.

**Configuration Flow:**
> Game Asset (JSON) -> Server Asset Loader -> **StaminaModule** (via LoadedAssetsEvent) -> SprintStaminaRegenDelay (Cache Invalidation)

**Gameplay Logic Flow:**
> Player Input (Sprint) -> Server Game Loop -> SprintStaminaEffectSystem -> Read/Write **SprintStaminaRegenDelay** on Entity -> Update EntityStatsModule data

