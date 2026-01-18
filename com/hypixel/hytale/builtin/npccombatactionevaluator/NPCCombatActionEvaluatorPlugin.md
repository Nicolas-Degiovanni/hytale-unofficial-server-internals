---
description: Architectural reference for NPCCombatActionEvaluatorPlugin
---

# NPCCombatActionEvaluatorPlugin

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator
**Type:** Singleton

## Definition
```java
// Signature
public class NPCCombatActionEvaluatorPlugin extends JavaPlugin {
```

## Architecture & Concepts
The NPCCombatActionEvaluatorPlugin serves as the central bootstrap and registration authority for the server-side NPC combat AI subsystem. It is not a runtime component that executes game logic; rather, it is an initialization-time module that configures the engine to support a sophisticated, data-driven combat evaluation framework.

Its primary responsibilities are:
1.  **ECS Extension:** It integrates the combat AI feature into the core Entity Component System (ECS). It achieves this by registering new, combat-specific component types such as TargetMemory, DamageMemory, and CombatActionEvaluator. These components store the runtime state for each individual NPC's combat awareness and decision-making process.
2.  **System Registration:** It registers the ECS Systems that contain the actual AI logic. Systems like EvaluatorTick and CollectDamage are registered to operate on their corresponding components during the main server tick loop. This decouples the AI logic from the NPC entity data itself, following standard ECS patterns.
3.  **Asset Definition:** It registers custom asset types with the HytaleAssetStore, most notably CombatActionOption. This allows game designers and modders to define specific NPC combat behaviors, attacks, and decision criteria in external data files (e.g., JSON), which are then loaded and utilized by the registered systems.
4.  **Behavior Tree Integration:** It registers builders for core components used within the higher-level NPC decision-making framework (e.g., Behavior Trees or State Machines). Components like BuilderSensorCombatActionEvaluator and BuilderActionCombatAbility act as nodes or building blocks that designers can use to construct complex AI behaviors.
5.  **Condition Registration:** It extends the set of available AI conditions by registering custom codecs for classes like RecentSustainedDamageCondition. This enables designers to create more nuanced AI triggers based on dynamic combat state.

In essence, this plugin wires together the asset system, the ECS, and the NPC behavior system to create a cohesive and extensible combat AI framework.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's plugin loading mechanism during server startup. The framework invokes the constructor and subsequently calls the setup method.
-   **Scope:** This object is a global singleton that persists for the entire lifetime of the server process. The static `instance` field is set within the setup method, providing a global access point.
-   **Destruction:** The object is destroyed when the server shuts down or if the plugin is explicitly unloaded by the server framework.

## Internal State & Concurrency
-   **State:** The plugin's state consists of several ComponentType fields. These are effectively immutable handles or identifiers assigned by the EntityStoreRegistry during the single-threaded setup phase. After initialization, the plugin's own state does not change. All mutable combat state is managed within the ECS components (e.g., TargetMemory) attached to individual NPC entities, not within the plugin itself.
-   **Thread Safety:** This class is thread-safe. The setup method, which mutates the internal state, is guaranteed by the framework to be called only once from a single thread during server initialization. All public getter methods and the static `get` method are safe to call from any thread as they return immutable, pre-initialized values. The registered ECS systems are responsible for their own concurrency guarantees within the engine's scheduler.

## API Surface
The primary contract of this class is established through its registration side-effects during setup. The public methods are accessors for the results of that registration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static NPCCombatActionEvaluatorPlugin | O(1) | Returns the global singleton instance of the plugin. Throws NullPointerException if called before setup is complete. |
| getTargetMemoryComponentType() | ComponentType | O(1) | Returns the registered handle for the TargetMemory component. |
| getCombatActionEvaluatorComponentType() | ComponentType | O(1) | Returns the registered handle for the CombatActionEvaluator component. |
| getCombatConstructionDataComponentType() | ComponentType | O(1) | Returns the registered handle for the CombatConstructionData component. |
| getDamageMemoryComponentType() | ComponentType | O(1) | Returns the registered handle for the DamageMemory component. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with this plugin during the game loop. Its primary use is to retrieve the component type handles for use with the ECS registry, often during the setup of other systems or for debugging purposes.

```java
// Example: A separate system needing to interact with TargetMemory
NPCCombatActionEvaluatorPlugin plugin = NPCCombatActionEvaluatorPlugin.get();
ComponentType<EntityStore, TargetMemory> memType = plugin.getTargetMemoryComponentType();

// Now use the type to get the component from an entity
EntityStore entityStore = ...;
EntityId npcId = ...;
TargetMemory memory = entityStore.get(npcId, memType);
if (memory != null) {
    // Process the NPC's target memory
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new NPCCombatActionEvaluatorPlugin()`. The plugin framework is the sole owner of its lifecycle. Doing so will bypass the engine's initialization sequence and result in a non-functional, detached object.
-   **Manual Setup:** Never call the `setup()` method directly. This will lead to double registration of all components, systems, and assets, which will critically fail and likely crash the server.
-   **Early Access:** Do not call `NPCCombatActionEvaluatorPlugin.get()` during the static initialization phase of another class. The plugin's `instance` may not have been set yet, leading to a NullPointerException. Access should only occur after the plugin loading phase is complete.

## Data Pipeline
This plugin does not process a stream of data itself. Instead, it **defines and enables** the data processing pipeline for the NPC combat AI, which executes every server tick.

> Flow:
> 1. **Design-Time:** A designer creates a `CombatActionOption` asset file (e.g., `heavy_attack.json`) defining conditions and outcomes.
> 2. **Server Startup:** The `AssetRegistry`, configured by this plugin, loads and parses the `CombatActionOption` asset.
> 3. **NPC Spawns:** The `CombatActionEvaluatorSystems.OnAdded` system runs, attaching and initializing the `CombatActionEvaluator` and other components on the new NPC entity.
> 4. **Runtime (Game Tick):**
>    - A player damages the NPC.
>    - The `DamageMemorySystems.CollectDamage` system detects this event and records it in the NPC's `DamageMemory` component.
>    - The `CombatActionEvaluatorSystems.EvaluatorTick` system executes for the NPC.
>    - The system reads the NPC's `TargetMemory` and `DamageMemory` components.
>    - It evaluates the conditions from all available `CombatActionOption` assets against the component state.
>    - A winning action is selected and executed, triggering an attack or behavior change.

