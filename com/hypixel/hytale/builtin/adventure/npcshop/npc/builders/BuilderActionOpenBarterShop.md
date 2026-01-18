---
description: Architectural reference for BuilderActionOpenBarterShop
---

# BuilderActionOpenBarterShop

**Package:** com.hypixel.hytale.builtin.adventure.npcshop.npc.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionOpenBarterShop extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionOpenBarterShop class is a server-side component within the NPC (Non-Player Character) asset definition framework. It functions as a *configurable factory* responsible for translating a static data definition, typically from a JSON file, into a live, executable game object.

Its primary role is to construct an `ActionOpenBarterShop` instance. This class acts as the critical bridge between the declarative world of game design assets and the imperative world of the server's runtime logic. When a game designer specifies in an NPC's behavior file that an interaction should open a shop, this builder is invoked by the asset loading system to parse that definition, validate it, and prepare the corresponding runtime action.

The use of an `AssetHolder` for the `shopId` is a key architectural choice. It decouples the NPC action from a hardcoded shop identifier, allowing the specific shop to be resolved dynamically based on the NPC's execution context at runtime. This enables the creation of generic, reusable NPC behaviors that can be configured to open different shops without modifying the core behavior logic.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderActionOpenBarterShop is created by the server's `NPCAssetLoader` or a similar system during the server bootstrap or a hot-reload event. It is instantiated specifically when the loader encounters an action of type "OpenBarterShop" within an NPC behavior asset file.
- **Scope:** The lifecycle of a builder instance is extremely short and confined to the asset parsing phase. It exists only to be configured via `readConfig` and then immediately used to produce an `Action` via its `build` method.
- **Destruction:** The builder object is eligible for garbage collection as soon as the `build` method returns. It holds no persistent references and is not intended to outlive the asset loading transaction.

## Internal State & Concurrency
- **State:** This class is **Mutable**. Its primary internal state is the `shopId` field, which is populated by the `readConfig` method. This state is transient and essential for the subsequent call to the `build` method.
- **Thread Safety:** This class is **Not Thread-Safe** and must not be accessed concurrently. It is designed to be used exclusively within the single-threaded context of the server's asset loading pipeline. Sharing an instance across threads will result in race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new `ActionOpenBarterShop` instance using the previously configured state. |
| readConfig(JsonElement) | BuilderActionOpenBarterShop | O(N) | Configures the builder from a JSON data structure. Validates that the required "Shop" asset exists. Throws if configuration is invalid. |
| getShopId(BuilderSupport) | String | O(1) | Resolves the configured shop identifier using the provided runtime execution context. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is invoked automatically by the engine's asset loading systems. The conceptual flow is as follows:

```java
// Conceptual example of engine-level usage
// This code does not exist in this form but illustrates the pattern.

JsonElement actionJson = parseNpcBehaviorFile(".../npc.json");
BuilderActionOpenBarterShop builder = new BuilderActionOpenBarterShop();

// The engine configures the builder from the asset file
builder.readConfig(actionJson);

// The engine builds the final runtime object
// The 'builder' instance is now discarded
Action runtimeAction = builder.build(builderSupport);
npc.addInteractionAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of this class manually with `new`. The NPC asset system is solely responsible for its lifecycle. Manually creating it bypasses the asset pipeline and will result in a non-functional action.
- **State Re-use:** Do not attempt to reuse a builder instance. Each builder is designed for a single configure-then-build operation. Caching or re-using instances will lead to incorrect NPC behaviors.
- **Manual Configuration:** Avoid calling `readConfig` outside of the asset loading pipeline. Bypassing the `BarterShopExistsValidator` by manually manipulating internal state can lead to runtime errors when the NPC attempts to open a non-existent shop.

## Data Pipeline
This component operates within a configuration-time data pipeline, not a real-time one. Its purpose is to transform declarative data into an executable object.

> Flow:
> NPC Behavior JSON Asset -> Server Asset Parser -> **BuilderActionOpenBarterShop::readConfig** -> **BuilderActionOpenBarterShop::build** -> `ActionOpenBarterShop` Instance -> Stored in NPC Behavior Tree

