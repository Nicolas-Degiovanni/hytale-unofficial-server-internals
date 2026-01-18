---
description: Architectural reference for ClearEntityEffectInteraction
---

# ClearEntityEffectInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class ClearEntityEffectInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The **ClearEntityEffectInteraction** class is a data-driven command object that encapsulates the server-side logic for removing a specific status effect from a target entity. It is not a service or manager, but rather a concrete implementation of the Strategy pattern, representing a single, configurable action within the Hytale Interaction System.

This class acts as a bridge between the high-level game configuration (defined in asset files) and the low-level Entity Component System (ECS). Its primary role is to be deserialized from a configuration file using its static **CODEC** field. Once loaded, it exists as a stateless, reusable template. When a corresponding in-game event occurs, the Interaction System invokes the **firstRun** method on this object, passing in the runtime context of the event.

Execution relies heavily on the server's ECS command buffer. Instead of directly modifying the target entity's components, it queues a command to remove the effect. This pattern ensures that all entity state modifications are deferred and executed at a safe, synchronized point in the main server tick, preventing race conditions and maintaining world state integrity.

## Lifecycle & Ownership
- **Creation:** Instances are not created programmatically using the *new* keyword. They are exclusively instantiated by the **BuilderCodec** during the server's asset loading phase. The server reads a configuration file (e.g., JSON) and uses the codec to deserialize the data into a **ClearEntityEffectInteraction** object.

- **Scope:** An instance represents a persistent interaction *definition*. It is loaded once at server startup and stored in a central registry or asset map. It remains in memory for the entire server session, acting as a flyweight object that can be reused for every occurrence of this specific interaction.

- **Destruction:** The object is garbage collected when the server shuts down and the asset registries are cleared. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** The internal state, consisting of **entityEffectId** and **entityTarget**, is populated once during deserialization. After this initial loading, the object's state is effectively **immutable**. It functions as a read-only configuration template.

- **Thread Safety:** This class is inherently thread-safe for read operations. The execution method, **firstRun**, is designed to be called exclusively from the server's main game thread. Concurrency safety for world state modification is not managed by this class but is guaranteed by the **CommandBuffer** system it integrates with.

    **WARNING:** Invoking **firstRun** from an asynchronous thread or worker pool will bypass the command buffer's protections and lead to world state corruption. All interaction logic must be dispatched to the main server tick.

## API Surface
The public contract is inherited from **SimpleInstantInteraction**. The core entry point is **firstRun**.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Executes the interaction. Locates the target entity and queues a command to remove the configured entity effect. This operation is deferred via the CommandBuffer. |

## Integration Patterns

### Standard Usage
This class is not intended for direct programmatic use. Its behavior is defined declaratively in external configuration files, which are then loaded by the server.

A typical definition within a game asset file might look like this conceptual example:

```json
// Example: An item's "onUse" effect configuration
{
  "onUseInteraction": {
    "type": "ClearEntityEffectInteraction",
    "EntityEffectId": "hytale:poison",
    "Entity": "USER"
  }
}
```

The server's Interaction System would then load this definition and execute it when the item is used.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ClearEntityEffectInteraction()`. This bypasses the critical deserialization and validation logic provided by the **CODEC** and will result in a non-functional object.

- **Manual Execution:** Do not call the **firstRun** method directly with a manually constructed **InteractionContext**. Doing so circumvents the server's main loop and command buffer, creating a high risk of concurrency violations and data corruption.

- **State Mutation:** Do not attempt to modify the **entityEffectId** or **entityTarget** fields after the object has been loaded. These are intended to be read-only configuration values.

## Data Pipeline

This component participates in two distinct data flows: a configuration pipeline at load time and an execution pipeline at runtime.

**Configuration Pipeline (Server Start):**
> Flow:
> Asset File (e.g., JSON) -> Server Asset Loader -> **BuilderCodec** -> In-Memory **ClearEntityEffectInteraction** Instance

**Execution Pipeline (In-Game Event):**
> Flow:
> Player Action -> Interaction System -> **ClearEntityEffectInteraction::firstRun** -> CommandBuffer -> EffectControllerComponent Update -> Network Packet Generation -> Client State Sync

