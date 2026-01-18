---
description: Architectural reference for CombatActionOption
---

# CombatActionOption

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.evaluator.combatactions
**Type:** Asset-Defined Component

## Definition
```java
// Signature
public abstract class CombatActionOption extends Option implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, CombatActionOption>> {
```

## Architecture & Concepts

The CombatActionOption class is the abstract foundation for all discrete combat behaviors available to NPCs. It represents a single, executable action that an NPC can choose, such as casting a spell, changing a stance, or selecting a target. This system is fundamentally data-driven; concrete actions are not instantiated directly in code but are defined in JSON asset files and loaded by the engine at startup.

Architecturally, this class serves as a bridge between the asset configuration and the server-side NPC decision-making logic. It inherits from the `Option` class, signaling its role within a selection system where an NPC evaluates multiple choices and picks the most suitable one.

The core of its design is a polymorphic deserialization pattern. The static `CODEC` field, an `AssetCodecMapCodec`, acts as a registry that maps string identifiers from JSON (e.g., "State", "Ability") to concrete Java subclasses like `StateCombatAction` and `AbilityCombatAction`. This allows designers to compose complex NPC behaviors entirely within data files without requiring engine code changes. All loaded actions are stored in a global, static `AssetStore`, making them readily available for lookup by any system.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale `AssetStore` during the server's asset loading phase. The `AssetCodecMapCodec` reads JSON definitions, determines the correct subclass based on a type key, and instantiates it, populating its fields from the data. Direct instantiation in game logic is a critical anti-pattern.
-   **Scope:** Application-scoped. Once an action is loaded from an asset, it is stored in the static `ASSET_STORE` and persists for the entire server session. These objects are treated as immutable definitions or templates.
-   **Destruction:** Instances are destroyed only when the `AssetRegistry` is cleared, which typically occurs on server shutdown or a full, engine-level asset reload. They are not garbage collected during normal gameplay.

## Internal State & Concurrency
-   **State:** Effectively immutable. All properties of a `CombatActionOption` are populated once during deserialization from JSON assets. These objects should be treated as read-only definitions at runtime. They do not contain per-NPC instance state; that is managed by components like the `ValueStore` passed into the `execute` method.
-   **Thread Safety:** The instances themselves are safe to read from multiple threads due to their immutable nature. However, the initialization of the static `ASSET_STORE` via the `getAssetStore` method is **not** thread-safe. It employs a lazy-initialization pattern that assumes it will be called for the first time from a single, controlled thread during server startup.

**WARNING:** Concurrent invocation of `getAssetStore` before the store is initialized will result in a race condition and is an unsupported operation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | abstract void | Varies | Executes the combat action. This is the primary entry point for applying the action's logic to the world state via the CommandBuffer. |
| isBasicAttackAllowed(...) | abstract boolean | O(1) | A predicate used by the decision-making system to determine if a basic attack can be performed concurrently with this action's effects. |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, static registry of all loaded CombatActionOption assets. |
| getId() | String | O(1) | Returns the unique asset identifier for this action. |
| cancelBasicAttackOnSelect() | boolean | O(1) | Returns true if selecting this option should interrupt any ongoing basic attack. |

## Integration Patterns

### Standard Usage
Developers should never create instances of `CombatActionOption` directly. The correct pattern is to retrieve a pre-loaded, immutable definition from the global `AssetStore` using its unique string identifier. The `CombatActionEvaluator` or a similar AI system then invokes methods on this shared instance.

```java
// Retrieve the globally-defined action from the asset system
IndexedLookupTableAssetMap<String, CombatActionOption> actions = CombatActionOption.getAssetMap();
CombatActionOption fireballAction = actions.get("npc_mage_fireball");

// The evaluator would then execute this action for a specific NPC
if (fireballAction != null) {
    // The 'index' and other parameters provide the per-NPC context
    fireballAction.execute(npcEntityIndex, chunk, commandBuffer, role, evaluator, valueStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new StateCombatAction()` or similar. This bypasses the asset pipeline, resulting in a partially initialized object that is not managed by the engine and will cause unpredictable behavior.
-   **Runtime State Modification:** Do not attempt to modify the fields of a `CombatActionOption` retrieved from the `AssetStore`. These are shared, global definitions. Modifying one would affect every NPC in the game that uses that action. Per-entity state must be stored in ECS components or the `ValueStore`.
-   **Concurrent Initialization:** Do not call `CombatActionOption.getAssetStore()` from multiple worker threads during server startup. This will lead to a race condition when creating the static `ASSET_STORE` instance.

## Data Pipeline
The flow of data from configuration to execution is a one-way pipeline managed by the engine's core systems.

> Flow:
> JSON Asset File -> AssetStore (Engine Loading) -> **AssetCodecMapCodec** (Deserialization) -> Populated **CombatActionOption** instance in static map -> CombatActionEvaluator (Lookup by ID) -> `execute` call -> CommandBuffer -> World State Change

