---
description: Architectural reference for BuilderRoleVariant
---

# BuilderRoleVariant

**Package:** com.hypixel.hytale.server.npc.role.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderRoleVariant extends SpawnableWithModelBuilder<Role> {
```

## Architecture & Concepts

The BuilderRoleVariant is a specialized builder that implements a form of configuration-level inheritance. It does not define a new Non-Player Character Role from scratch. Instead, it acts as a proxy or decorator for an existing Role definition, applying a set of modifications to it at runtime.

This component is central to the server's data-driven NPC system, allowing designers to create numerous variations of a base NPC archetype without duplicating large blocks of JSON configuration. For example, a base *Goblin* role can be defined once, and then variants like *Goblin Archer* or *Goblin Shaman* can be created using BuilderRoleVariant to override specific properties like equipment, combat behavior, or model.

The core architectural pattern is **delegation with context modification**. When a method like build or canSpawn is invoked on a BuilderRoleVariant, it performs the following critical sequence:

1.  **Lookup:** It retrieves the referenced "super role" builder from the central BuilderManager cache using a pre-calculated integer index.
2.  **Scope Composition:** It uses its internal BuilderModifier to create a new, temporary Scope. This new scope contains the overridden variables and parameters defined in the variant's configuration. If the super role is also a variant, this process is chained, creating a final, composite scope.
3.  **Context Switch:** It temporarily replaces the current Scope in the active ExecutionContext with the newly composed one.
4.  **Delegation:** It invokes the corresponding method (e.g., build) on the super role builder. The super role builder now executes within the context of the modified scope, using the variant's values where present.
5.  **Context Restoration:** It restores the original Scope to the ExecutionContext, ensuring the modification is isolated and does not leak.

This mechanism allows for powerful and efficient composition of entity behaviors, where the final state of an NPC is determined by a layered application of configuration scopes.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the server's asset loading system, specifically the BuilderManager, during the parsing of NPC JSON configuration files. Each JSON object that defines a role variant results in the instantiation of one BuilderRoleVariant object.
- **Scope:** The object's lifetime is tied to the asset loading cycle. It is stored in the BuilderManager's cache and persists as long as the current set of NPC configurations is considered valid.
- **Destruction:** The object is marked for garbage collection when a server asset reload occurs. The `markNeedsReload` method signals to the NPCPlugin that this builder's data is stale, eventually leading to the disposal of the old cache and its contents.

## Internal State & Concurrency
- **State:** The object is stateful but becomes effectively immutable after the initial `readConfig` phase. Its primary state consists of `referenceIndex` (an integer pointing to the base builder) and `modifier` (an object containing the configuration overrides). This state is not intended to be mutated after initialization.
- **Thread Safety:** This class is **not** internally synchronized. It is designed to be used in a single-threaded context during configuration loading. Runtime methods like `build` and `canSpawn` are frequently called from multiple game threads. Safety is achieved by manipulating the `ExecutionContext`, which is assumed to be a thread-local construct.

**WARNING:** Concurrent calls to configuration methods like `readConfig` will result in undefined behavior. Runtime methods rely entirely on the thread-safety of the passed-in `ExecutionContext` and the immutability of the builder's internal state post-initialization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Role | O(super.build) | Delegates Role instantiation to the referenced builder within a modified execution context. Returns null if the base builder fails or is not spawnable. |
| readConfig(JsonElement) | Builder<Role> | O(N) | Parses the variant's JSON definition. Populates the internal reference and modifier state. This is the primary initialization vector. |
| validate(...) | boolean | O(super.validate) | Delegates validation logic to the referenced builder within a modified scope. |
| canSpawn(SpawningContext) | SpawnTestResult | O(super.canSpawn) | Determines if the NPC variant can spawn by delegating the check to its base builder under the variant's conditions. |
| createModifierScope(ExecutionContext) | Scope | O(D) | Recursively constructs a new Scope object by chaining all modifiers from this variant up to the root non-variant builder. D is the depth of the variant chain. |

## Integration Patterns

### Standard Usage
A developer or designer does not interact with this class directly in Java code. Instead, they define a variant in a JSON asset file. The system uses this builder transparently to realize the configuration.

**Example NPC Role Variant Definition (npc/goblin_archer.json):**
```json
{
  "$Label": "BuilderRoleVariant",
  "Reference": "hytale:npc/goblin_base",
  "Modify": {
    "Parameters": {
      "mainHandItem": "hytale:bow"
    },
    "CombatConfig": "hytale:combat/ranged_coward"
  }
}
```
The game engine then uses this builder to spawn a *Goblin Archer* by applying the specified modifications to the *Goblin Base* definition at runtime.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderRoleVariant()`. The object is deeply integrated with the BuilderManager for dependency resolution and caching. Instantiating it manually will result in a non-functional object and likely throw NullPointerExceptions.
- **Circular References:** Defining variants that reference each other in a loop (e.g., A references B, and B references A) is a critical anti-pattern. While the system may have safeguards, this can lead to StackOverflowErrors during scope creation or validation.
- **State Mutation:** Do not attempt to modify the internal state of a BuilderRoleVariant after it has been initialized by the asset loader. The system relies on its post-load immutability for predictable behavior.

## Data Pipeline

The BuilderRoleVariant operates in two distinct phases: a configuration-time pipeline and a runtime execution pipeline.

**Configuration-Time Flow:**
> NPC Role JSON File -> Server Asset Loader -> **BuilderRoleVariant.readConfig()** -> Cached BuilderRoleVariant Instance in BuilderManager

**Runtime Execution Flow (Example: Spawning):**
> Spawner Request -> `build(support)` on **BuilderRoleVariant** -> `executeOnSuperRole()` -> Create Modified Scope -> `build(support)` on Super Role Builder -> Final Role Instance

