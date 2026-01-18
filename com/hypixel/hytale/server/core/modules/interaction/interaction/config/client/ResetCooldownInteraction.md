---
description: Architectural reference for ResetCooldownInteraction
---

# ResetCooldownInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ResetCooldownInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The ResetCooldownInteraction class is a data-driven command object within the server's Interaction System. It represents a single, instantaneous action: resetting a named cooldown timer for an entity. This class is not intended for direct, imperative use by developers. Instead, it is deserialized from game configuration files (e.g., JSON) via its static `CODEC`.

Architecturally, it serves as a declarative instruction within a larger `InteractionChain`. When an entity performs an action, the `InteractionManager` processes the corresponding chain, and upon reaching a ResetCooldownInteraction node, executes its logic.

Its primary function is to interface with the stateful `CooldownHandler` service. It reads its own configuration, determines the final cooldown parameters based on a clear precedence hierarchy, and instructs the `CooldownHandler` to reset the state of a specific cooldown timer. This decouples the definition of an interaction (data) from the core game state management (the `CooldownHandler` service).

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the `BuilderCodec` system during server startup or when game configurations are loaded. It is a critical anti-pattern to instantiate this class directly using the `new` keyword, as this bypasses essential configuration.
- **Scope:** The object's lifetime is bound to its parent `InteractionChain`. It is a transient object, existing only as a configured node within a larger interaction graph. It does not persist beyond the scope of its parent configuration.
- **Destruction:** The object is managed by the Java garbage collector. There are no manual cleanup or `close` methods. It is reclaimed once the `InteractionChain` that owns it is unloaded.

## Internal State & Concurrency
- **State:** The class holds a single, nullable `InteractionCooldown` field. This state is populated during deserialization by the `CODEC` and is treated as **immutable** for the object's entire lifecycle. The `firstRun` method reads this state but does not modify it.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. The entire Interaction System is designed to be executed sequentially within the server's primary game loop. All state modification is delegated to the `CooldownHandler`, which is responsible for its own concurrency guarantees.

## API Surface
The public API is minimal, as the class is primarily invoked by the interaction system itself.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(1) | **Entry Point.** Executes the cooldown reset logic. This method should only be called by the `InteractionManager`. |
| generatePacket() | Interaction | O(1) | Creates the corresponding network packet to inform the client of the action. |
| configurePacket(packet) | void | O(1) | Populates the network packet with data from this object's internal state. |

## Integration Patterns

### Standard Usage
This class is not used in procedural code. It is defined declaratively within a game data file, such as an interaction JSON configuration. The engine's `InteractionManager` is responsible for finding and executing it.

A conceptual configuration might look like this:

```json
// Example: in-game item_actions.json
{
  "id": "use_health_potion",
  "chain": [
    {
      "type": "Hytale:ApplyEffect",
      "effect": "Hytale:Heal"
    },
    {
      "type": "Hytale:ResetCooldownInteraction",
      "cooldown": {
        "cooldownId": "potion_cooldown"
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ResetCooldownInteraction()`. The object will be in an invalid state and will cause `NullPointerException`s when executed. Always define interactions in configuration files.
- **Manual Execution:** Do not call the `firstRun` method directly. This bypasses the state management and context provided by the `InteractionManager` and `InteractionChain`, leading to unpredictable behavior.
- **State Modification:** Do not attempt to modify the `cooldown` field after the object has been created by the codec. The system treats this object as immutable post-creation.

## Data Pipeline
The flow of data and control for this component is strictly one-way, from configuration to stateful execution.

> Flow:
> Game Configuration File (JSON) -> Server Asset Loader -> **BuilderCodec** -> In-Memory **ResetCooldownInteraction** Object -> `InteractionManager` executes chain -> `firstRun()` method is called -> `CooldownHandler` state is mutated.

