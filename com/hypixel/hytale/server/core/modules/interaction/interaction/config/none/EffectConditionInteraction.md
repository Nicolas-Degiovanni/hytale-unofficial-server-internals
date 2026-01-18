---
description: Architectural reference for EffectConditionInteraction
---

# EffectConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class EffectConditionInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts

The EffectConditionInteraction is a server-side, data-driven logic component that functions as a conditional gate within a larger interaction sequence. It extends SimpleInstantInteraction, signifying that its execution is immediate and does not involve a persistent state or duration.

Its primary architectural role is to enforce game rules by checking the status effects on a target entity. An interaction chain can be configured to execute this check, and the chain will only proceed if the target entity either possesses a specific set of effects (**Match.All**) or is free of them (**Match.None**).

This class is designed to be configured entirely through external asset files, typically JSON. The static CODEC field defines the schema for this configuration, allowing game designers to create complex conditional behaviors without writing or modifying Java code. At runtime, the Interaction Module invokes this component's logic, making it a critical bridge between declarative game design and the server's entity-component-system.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the engine's serialization system via the static **CODEC** field. This process occurs when the server loads game assets, such as item definitions or skill configurations. It is never instantiated directly using the *new* keyword in gameplay code. The `afterDecode` hook within the CODEC is a critical part of its initialization, resolving string-based effect IDs into high-performance integer indexes.
- **Scope:** An instance of EffectConditionInteraction is effectively a stateless template. Its lifetime is tied to the asset that defines it. It persists in memory for the entire server session, ready to be executed by any number of game events.
- **Destruction:** The object is managed by the Java garbage collector. It is destroyed only when its defining asset is unloaded, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
- **State:** This object is **effectively immutable** after its creation by the codec. All its configuration fields, such as `entityEffectIds` and `match`, are set once during deserialization. The `entityEffectIndexes` array is a derived, cached value populated during the `afterDecode` lifecycle event to optimize runtime performance by avoiding repeated string lookups.
- **Thread Safety:** The class is **thread-safe for execution**. Because its internal state is immutable post-creation, a single shared instance can be safely referenced and executed by multiple threads simultaneously (e.g., different world threads processing player actions). The `firstRun` method operates on a thread-local `InteractionContext` and `CommandBuffer`, ensuring that its execution logic does not create race conditions.

## API Surface

The public API is minimal, as the class is primarily controlled by the interaction framework. The core logic is encapsulated in the protected `firstRun` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | **Framework-Internal.** Executes the core condition check. N is the number of effects in the configuration. Modifies the `InteractionContext` state to `Failed` if the condition is not met. |
| generatePacket() | Interaction | O(1) | **Framework-Internal.** Creates a network packet to inform the client about this interaction. |
| configurePacket(packet) | void | O(1) | **Framework-Internal.** Populates the network packet with the interaction's configuration data. |

## Integration Patterns

### Standard Usage

This class is not intended to be used directly in Java code. Instead, it is configured declaratively within a game asset file. The server's Interaction Module reads this configuration and invokes the interaction automatically.

The following conceptual JSON snippet illustrates how a game designer would configure this interaction:

```json
// Example: Part of an item's "onUse" interaction chain
{
  "type": "EffectCondition",
  "Entity": "TARGET",
  "EntityEffectIds": [
    "hytale:poison",
    "hytale:weakness"
  ],
  "Match": "All"
}
```

The system processes this as follows:
1. The `CODEC` deserializes the JSON into an `EffectConditionInteraction` object.
2. The `afterDecode` hook resolves the `EntityEffectIds` into integer indexes.
3. When the interaction is triggered, the framework calls `firstRun`.
4. The `firstRun` method checks if the entity targeted by the interaction has both the poison and weakness effects. If not, it sets the interaction state to `Failed`, halting the chain.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new EffectConditionInteraction()`. The object will be uninitialized and will cause `NullPointerException`s. Always define it in a data file to be loaded by the server's `CODEC` system.
- **State Mutation:** Do not attempt to modify the fields of a shared instance after it has been loaded. This violates its immutable design and will lead to unpredictable and inconsistent behavior across the server.
- **Ignoring Context:** The logic relies entirely on the `InteractionContext` passed into `firstRun`. Attempting to execute it without a valid context provided by the framework will fail.

## Data Pipeline

The data for this component follows two distinct pipelines: a configuration pipeline at load-time and an execution pipeline at run-time.

**Configuration Pipeline:**
> Asset File (JSON) → Server Asset Loader → **EffectConditionInteraction.CODEC** → `afterDecode` (Resolves IDs to Indexes) → In-Memory Instance

**Execution Pipeline:**
> Player Input → Interaction Module → **EffectConditionInteraction.firstRun(context)** → Reads `EffectControllerComponent` from Target Entity → Mutates `context.getState()` → Interaction Chain Continues or Halts

