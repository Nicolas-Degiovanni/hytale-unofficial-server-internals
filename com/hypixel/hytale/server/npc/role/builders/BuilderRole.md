---
description: Architectural reference for BuilderRole
---

# BuilderRole

**Package:** com.hypixel.hytale.server.npc.role.builders
**Type:** Transient (Builder)

## Definition
```java
// Signature
public class BuilderRole extends SpawnableWithModelBuilder<Role> implements SpawnEffect {
```

## Architecture & Concepts
The BuilderRole class is the central factory and configuration blueprint for all server-side Non-Player Characters (NPCs). It operates as a high-level aggregator within the NPC asset pipeline, translating static JSON configuration files into live, in-game Role objects. Its primary function is to deserialize, validate, and hold the complete behavioral definition of an NPC, including its stats, appearance, AI, movement, and interactions.

This class is not a simple data container. It orchestrates the construction of complex, interconnected subsystems:
*   **AI & Behavior:** It configures and links together StateTransitionControllers, Instructions, and StateEvaluators which form the core of an NPC's decision-making logic.
*   **Movement:** It manages a map of available MotionControllers, defining how an NPC navigates the world under different circumstances.
*   **Combat & Interaction:** It sets up combat parameters via BuilderCombatConfig and defines interaction variables, attitudes, and inventory.

A key architectural pattern employed is the use of *Holder* objects, such as AssetHolder, IntHolder, and BooleanHolder. These wrappers defer the final resolution of a configuration value until runtime. This allows NPC properties to be dynamic, potentially influenced by a runtime ExecutionContext that could include balancing variables or world state. The BuilderRole holds the raw, unevaluated configuration, while the resulting Role object queries it at runtime to get concrete values.

## Lifecycle & Ownership
- **Creation:** A BuilderRole instance is created by the server's BuilderManager during the asset loading phase at startup. One unique BuilderRole object is instantiated for each distinct NPC role defined in the game's JSON asset files (e.g., one for `hytale:wolf`, one for `hytale:goblin_warrior`).
- **Scope:** An instance persists for the entire server session. It is a cached, canonical representation of an NPC type's configuration. It is not created per-NPC-entity but is instead a shared factory.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down or when its corresponding asset is hot-reloaded, a process hinted at by the markNeedsReload method.

## Internal State & Concurrency
- **State:** The internal state of BuilderRole is highly mutable during the initial asset loading phase, specifically within the `readConfig` method call. After this initial parsing, the object's state should be considered immutable. All subsequent interactions are read-only operations, typically through its numerous getter methods.

- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** All methods are designed to be called from the main server thread.
    - The `readConfig` and `validate` methods are executed synchronously during the server's single-threaded startup procedure.
    - The `build` method and all data-retrieval getters are expected to be called from the main game loop during NPC spawning and updates.
    - **WARNING:** Concurrent access from multiple threads without external locking will lead to race conditions and unpredictable server state.

## API Surface
The public API is primarily concerned with the build lifecycle: deserialization, validation, and final object construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | BuilderRole | O(N) | Deserializes a JSON object into the builder's internal fields. N is the number of properties in the JSON. |
| build(BuilderSupport) | Role | O(1) | Constructs and returns a new Role instance, which maintains a reference to this builder for configuration data. |
| validate(...) | boolean | O(N) | Performs deep validation of the loaded configuration, checking for asset existence, logical consistency, and valid ranges. |
| canSpawn(SpawningContext) | SpawnTestResult | O(1) | Determines if an NPC with this role can spawn in a given context, checking conditions like breathability. |

## Integration Patterns

### Standard Usage
A developer or system should never instantiate BuilderRole directly. It is always retrieved from the central NPCPlugin or BuilderManager, which manages the asset cache. The builder is then used to create a Role instance, which is the component used by the live NPC entity.

```java
// 1. Retrieve the pre-loaded builder for a specific NPC type
BuilderRole wolfBuilder = NPCPlugin.get().getRoleBuilder("hytale:wolf");

// 2. Create a support context (contains ExecutionContext, etc.)
BuilderSupport support = createBuilderSupportForNewEntity();

// 3. Build the live Role object
Role wolfRole = wolfBuilder.build(support);

// 4. Assign the Role to a new game entity
newlySpawnedEntity.setRole(wolfRole);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderRole()`. This bypasses the asset loading pipeline, resulting in an unconfigured and useless object that is not registered with the game's systems.

- **Post-Load Modification:** Do not attempt to modify the state of a BuilderRole instance after the server has finished loading assets. This can lead to inconsistent behavior between newly spawned NPCs and existing ones of the same type.

- **Cross-Thread Access:** Do not share BuilderRole instances across threads or invoke its methods from asynchronous tasks without explicit, robust synchronization. The class is designed for single-threaded access from the main game thread.

## Data Pipeline
The BuilderRole is a critical stage in the NPC data pipeline, transforming static data into a usable runtime object.

> Flow:
> NPC Role JSON File -> Server Asset Loader -> **BuilderRole.readConfig()** -> In-Memory Builder Cache -> Spawning System -> **BuilderRole.build()** -> Live Role Object -> Attached to Game Entity

