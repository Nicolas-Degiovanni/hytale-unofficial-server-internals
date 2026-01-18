---
description: Architectural reference for DamageModule
---

# DamageModule

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Singleton

## Definition
```java
// Signature
public class DamageModule extends JavaPlugin {
```

## Architecture & Concepts
The DamageModule is a foundational server plugin that orchestrates the entire lifecycle of damage calculation and application within the Entity Component System (ECS). It does not contain damage logic itself; instead, it establishes a rigid, multi-stage pipeline and registers dozens of discrete systems that execute within that pipeline.

Its primary architectural contribution is the definition of three sequential **SystemGroup**s, which form a processing pipeline for any damage event:

1.  **gatherDamageGroup:** The initial stage where systems create and submit damage instances. Systems in this group are responsible for sourcing damage, such as from falling, environmental hazards, or direct attacks.
2.  **filterDamageGroup:** The second stage where damage instances are inspected, modified, or cancelled. Systems here apply armor reductions, invulnerability rules, world configuration settings, and other filtering logic.
3.  **inspectDamageGroup:** The final stage where systems react to the finalized damage amount without altering it. This is used for triggering side-effects like particle effects, sounds, hit animations, and UI updates.

This pipeline is enforced by an internal barrier system, **OrderGatherFilter**, which ensures that all gathering systems complete before any filtering systems begin. The module acts as a central registry and setup coordinator for all core logic related to damage, death, knockback, and respawning.

### Lifecycle & Ownership
- **Creation:** The DamageModule is instantiated once by the server's plugin loader during the bootstrap sequence. Its constructor is invoked by the framework, which supplies a **JavaPluginInit** context. The static singleton instance is set at this time.
- **Scope:** The instance persists for the entire runtime of the server. It is a global, session-scoped singleton.
- **Destruction:** The object is dereferenced and eligible for garbage collection only during server shutdown when the plugin system is unloaded.

## Internal State & Concurrency
- **State:** The internal state of the DamageModule consists of references to ECS constructs, primarily the **ComponentType**s and **SystemGroup**s it registers. This state is populated during the **setup** method and is considered immutable thereafter. The module itself is stateless regarding game-world data.
- **Thread Safety:** The class is thread-safe. Initialization occurs on the main server thread during startup. Post-initialization, its fields are read-only. Access via the static **get** method is safe from any thread, as it returns the pre-initialized singleton instance.

**WARNING:** While the module itself is thread-safe, the ECS systems it registers are executed by the engine's scheduler and must adhere to the engine's concurrency model. Do not assume systems can be invoked from arbitrary threads.

## API Surface
The public API is minimal, designed exclusively to expose the core pipeline stages (SystemGroups) to other systems and plugins.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | DamageModule | O(1) | Retrieves the static singleton instance of the module. Throws if accessed before plugin initialization. |
| getDeathComponentType() | ComponentType | O(1) | Returns the registered type for the DeathComponent. |
| getDeferredCorpseRemovalComponentType() | ComponentType | O(1) | Returns the registered type for the DeferredCorpseRemoval component. |
| getGatherDamageGroup() | SystemGroup | O(1) | Returns the SystemGroup for the first stage of the damage pipeline (gathering). |
| getFilterDamageGroup() | SystemGroup | O(1) | Returns the SystemGroup for the second stage of the damage pipeline (filtering). |
| getInspectDamageGroup() | SystemGroup | O(1) | Returns the SystemGroup for the final stage of the damage pipeline (inspection). |

## Integration Patterns

### Standard Usage
The primary integration pattern is not to call methods on the DamageModule, but to create a custom ECS system that declares a dependency on one of its SystemGroups. This injects custom logic directly into the damage pipeline.

```java
// Example: A custom system that grants resistance to magic damage.
// This system must be registered with the EntityStoreRegistry in your own plugin.

public class MagicResistanceSystem implements ISystem<EntityStore> {
    @Nonnull
    @Override
    public Set<Dependency<EntityStore>> getDependencies() {
        // This dependency ensures our system runs inside the damage filtering stage.
        return Set.of(new SystemGroupDependency<>(
            Order.IN,
            DamageModule.get().getFilterDamageGroup()
        ));
    }

    @Override
    public void run(@Nonnull EntityStore store, @Nonnull Query query) {
        // System logic to find and reduce incoming magic damage on entities.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new DamageModule()**. The server's plugin loader is solely responsible for its creation. Attempting to create it manually will break the singleton pattern and result in an uninitialized, non-functional object.
- **Premature Access:** Do not call **DamageModule.get()** from a plugin's constructor or during static initialization of another class. The singleton instance is only guaranteed to be available after the plugin's **setup** phase has begun.
- **Modifying Core Systems:** Do not attempt to unregister or modify the core systems registered by the DamageModule. This will destabilize the entire combat and health system. Extend its functionality via the provided SystemGroups.

## Data Pipeline
The module establishes a strict, sequential data flow for processing damage. An event or component representing potential damage enters the pipeline and is transformed at each stage.

> Flow:
> Damage Source (e.g., Fall, Attack) -> **gatherDamageGroup** (Systems create Damage components) -> **filterDamageGroup** (Systems modify or cancel Damage components) -> **inspectDamageGroup** (Systems read final Damage and trigger effects) -> Final Health/State Update

