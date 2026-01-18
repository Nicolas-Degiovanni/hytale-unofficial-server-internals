---
description: Architectural reference for RunRootInteraction
---

# RunRootInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Configuration Model

## Definition
```java
// Signature
public class RunRootInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The RunRootInteraction class is a fundamental component within the server's Interaction System. It does not implement any direct game logic itself; instead, it functions as a **stateless redirect** or **pointer**. Its sole purpose is to terminate the current interaction flow and immediately trigger a new, separate interaction tree defined by a RootInteraction.

This class is designed to be defined in configuration files (e.g., JSON assets) and deserialized at runtime by the engine's codec system. It enables game designers to chain complex interaction sequences together in a modular and reusable fashion without writing procedural code. For example, a simple button press interaction could use a RunRootInteraction to trigger a complex cinematic sequence, a quest dialogue, or a crafting menu, each defined in its own RootInteraction asset.

It is a concrete implementation of the Command design pattern, where its command is to delegate execution to another, more complex command object located via a string identifier.

### Lifecycle & Ownership
- **Creation:** RunRootInteraction instances are not created programmatically using the *new* keyword. They are exclusively instantiated by the Hytale **Codec** system during server initialization or when game assets are loaded. The static CODEC field defines the deserialization logic from a configuration source.
- **Scope:** An instance of this class is an immutable template that persists for the entire server session once loaded. It represents a piece of configured game logic, not a runtime state.
- **Destruction:** Instances are garbage collected when the server shuts down or performs a hot-reload of its configuration assets. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, specifically the target *rootInteraction* string, is set once during deserialization by the CODEC. It is never modified during the object's lifetime. This makes the object a reusable, stateless template.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability, a single instance can be safely referenced and used by multiple game loop threads simultaneously without risk of data corruption. The *firstRun* method is re-entrant, as all of its stateful operations are performed on the supplied InteractionContext object, which is assumed to be thread-local to the interaction being processed.

## API Surface
The primary API for this class is its configuration schema, defined by the static CODEC. The methods are part of a protected framework contract and are not intended for direct invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | Framework-invoked method. Immediately marks the current interaction as finished and commands the context to execute the configured RootInteraction. |
| generatePacket() | Interaction | O(1) | Framework-invoked method. Creates the network packet to inform the client of this interaction. |
| configurePacket(packet) | void | O(1) | Framework-invoked method. Populates the network packet with the ID of the target RootInteraction. |

## Integration Patterns

### Standard Usage
This component is not used in Java code directly. Instead, it is declared within a data file (e.g., a JSON asset) that defines an entity's or block's behavior.

```json
// Example: Defining an interaction on a block
{
  "onInteract": {
    "type": "RunRootInteraction",
    "RootInteraction": "hytale:quests/begin_wizard_tower"
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new RunRootInteraction()`. This bypasses the critical CODEC-based deserialization and validation process, resulting in a non-functional object that will cause NullPointerExceptions. Always define interactions in configuration files.
- **Direct Method Invocation:** Do not call `firstRun` or other methods on this class directly. These methods are part of a strict contract with the server's Interaction Module, which manages the InteractionContext and state machine. Calling them manually will lead to state corruption and unpredictable game behavior.
- **Post-Creation Modification:** Do not attempt to modify the `rootInteraction` field after the object has been created. The system relies on these objects being immutable templates.

## Data Pipeline
The primary flow for this component is from configuration data into a runtime game event.

> Flow:
> JSON Asset File -> Server Asset Loader -> **RunRootInteraction.CODEC** -> In-Memory RunRootInteraction Object -> Player Action -> Interaction Module executes `firstRun` -> InteractionContext is commanded to start a new RootInteraction.

