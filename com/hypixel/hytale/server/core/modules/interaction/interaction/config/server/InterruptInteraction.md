---
description: Architectural reference for InterruptInteraction
---

# InterruptInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient / Data Object

## Definition
```java
// Signature
public class InterruptInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The InterruptInteraction class is a concrete, data-driven action within the server's Interaction System. It is not a manager or a service, but rather a specific, configurable behavior that can be attached to items, abilities, or other game events. Its sole purpose is to forcefully terminate one or more ongoing interactions on a target entity.

Architecturally, this class embodies Hytale's data-driven design philosophy. Instances are not created programmatically via a constructor in game logic. Instead, they are deserialized from configuration files (e.g., JSON or HOCON) at server startup using the static CODEC field. This allows designers to create complex interruption logic without writing any Java code.

When triggered, its logic is executed within the server's main game loop. It operates by acquiring the target entity's InteractionManager component—the authority for all of that entity's interactions—and commanding it to cancel specific interaction chains based on a set of configurable filters, such as interaction type or entity tags. All entity state modifications are funneled through a CommandBuffer, ensuring that changes are applied safely at the end of the current tick, which prevents race conditions and maintains state consistency.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the framework's codec system during the asset loading phase. The static `CODEC` field is responsible for deserializing a configuration block from a game data file into a fully-formed InterruptInteraction object. The `afterDecode` hook is critical for post-processing, such as resolving string-based tags into more performant integer indexes via the AssetRegistry.
-   **Scope:** An instance of InterruptInteraction is a stateless template that persists as long as its defining configuration is loaded by the server. It is typically part of a larger configuration graph (e.g., an item's definition) and is reused for every execution of that specific interaction.
-   **Destruction:** The object is eligible for garbage collection when the server unloads the asset pack or module that contains its definition. This typically occurs during a server shutdown or a hot-reload of game content.

## Internal State & Concurrency
-   **State:** The object's state is configured at load time and is **effectively immutable** afterward. Fields like `entityTarget`, `interruptTypes`, and tag-related strings are set once by the CODEC. The integer-based tag indexes (`requiredTagIndex`, `excludedTagIndex`) are derived and cached during the `afterDecode` step for performance. The `firstRun` method only reads this state; it does not modify it.
-   **Thread Safety:** The object itself is thread-safe for reads due to its immutable nature post-creation. However, the primary execution method, `firstRun`, is **not thread-safe** and is designed to be executed exclusively on the server's main game thread as part of the tick processing cycle. It relies on a thread-local `InteractionContext` and queues its work in a `CommandBuffer`, which is the designated pattern for safe state mutation within the engine's architecture.

## API Surface
The primary contract is the `firstRun` method, inherited from its parent. The static `CODEC` field is the entry point for the framework, not for general-purpose use.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the interruption logic. N is the number of active interaction chains on the target entity. This is the core operational method called by the InteractionModule. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly in Java. Instead, they define its behavior declaratively in a data file. The system then loads this configuration and executes it when the corresponding game event occurs.

A conceptual configuration file might look like this:
```json
// In an item or ability definition file
"onUse": {
  "type": "InterruptInteraction",
  "Entity": "TARGET", // Target the entity the user is looking at
  "InterruptTypes": [
    "CHANNELING",
    "CHARGING"
  ],
  "RequiredTag": "spellcasting"
}
```
The engine's InteractionModule is responsible for parsing this data, creating an InterruptInteraction instance, and invoking its `firstRun` method at the appropriate time.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new InterruptInteraction()`. This bypasses the CODEC, leaving the object in an invalid state. Specifically, the critical tag indexes (`requiredTagIndex`, `excludedTagIndex`) will not be resolved from their string counterparts, causing the filtering logic to fail.
-   **External Invocation:** Do not call the `firstRun` method from outside the server's core InteractionModule. The method depends on a valid `InteractionContext` and `CommandBuffer`, which are only guaranteed to be correctly configured during the engine's standard interaction processing phase.
-   **Post-Creation Mutation:** Do not modify the public fields of this class after it has been decoded. These are considered static configuration and changing them at runtime will lead to unpredictable behavior for all subsequent uses of this interaction.

## Data Pipeline
The flow of data and control for this component is linear and occurs within a single server tick.

> Flow:
> Player Input or Game Event -> InteractionModule resolves the action to an **InterruptInteraction** instance -> The module invokes `firstRun` with the current `InteractionContext` -> `firstRun` retrieves the target entity and its `InteractionManager` via a `CommandBuffer` -> The method iterates active `InteractionChain`s on the manager, applying type and tag filters -> For each match, a `cancelChains` command is queued on the `InteractionManager` -> The `CommandBuffer` flushes at the end of the tick, applying the cancellation.

