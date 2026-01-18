---
description: Architectural reference for ConditionInteraction
---

# ConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Configuration Model

## Definition
```java
// Signature
public class ConditionInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The ConditionInteraction class is a server-side data model that functions as a conditional gate within the broader Interaction System. It represents a specific, stateless rule set that must be satisfied for an interaction to be considered successful. This class is not intended for direct programmatic use but is instead instantiated and configured by the Hytale Codec system from game asset files.

Architecturally, it acts as a leaf node in an interaction graph. When a player triggers an action, the server's Interaction Module processes a chain of these interaction objects. The ConditionInteraction's role is to inspect the current state of the acting entity—specifically its `Player` and `MovementStatesComponent`—and compare it against its configured preconditions.

Its core design principles are:
*   **Data-Driven:** Behavior is defined entirely by deserialized data, not by code. The static `CODEC` field dictates the available configuration keys (e.g., `RequiredGameMode`, `Jumping`, `Flying`).
*   **Stateless Evaluation:** An instance of ConditionInteraction holds no state related to a specific interaction event. It is a reusable set of rules that can be evaluated against any number of entities without side effects.
*   **Component-Oriented:** It operates directly on the server's Entity-Component-System (ECS) data, reading state from components like `MovementStatesComponent` via a `CommandBuffer` to ensure data consistency within a single server tick.

## Lifecycle & Ownership
- **Creation:** ConditionInteraction instances are created exclusively by the Hytale `BuilderCodec` system during server startup or when game assets are hot-loaded. The server reads a configuration file (e.g., a JSON or HOCON file defining an item's behavior) and uses the static `CODEC` definition to deserialize the data into a new ConditionInteraction object.

- **Scope:** The object's lifetime is tied to the asset that defines it. It persists in memory as part of a larger configuration object graph and is reused every time the corresponding game interaction is triggered. It is effectively a singleton from the perspective of a specific game rule.

- **Destruction:** The object is eligible for garbage collection only when the server unloads the associated game assets, such as during a server shutdown or a full asset reload.

## Internal State & Concurrency
- **State:** The internal state of a ConditionInteraction object consists of its configured conditions (`requiredGameMode`, `jumping`, `swimming`, etc.). This state is set once upon deserialization and is **treated as immutable** for the entire lifetime of the object. It does not cache any runtime data.

- **Thread Safety:** This class is inherently thread-safe. Its state is immutable post-creation, and the primary `tick0` method is a pure function with respect to the object itself. It reads data from the `InteractionContext` and its associated components but does not modify its own fields. It is designed to be executed within the server's single-threaded game loop for a given world or entity, and its reliance on the `CommandBuffer` pattern reinforces this design.

## API Surface
The primary programmatic interface is the `tick0` method, which is invoked by the server's Interaction Module. The *effective* public API for game designers is the set of keys defined in the static `CODEC`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | protected void | O(1) | Evaluates the entity's state against the configured conditions. Sets the `InteractionContext` state to `Finished` or `Failed`. This is a framework-internal method. |
| CODEC | public static final BuilderCodec | N/A | Defines the deserialization schema. This is the primary configuration interface for game designers. |

## Integration Patterns

### Standard Usage
This class is not used by writing Java code. Instead, a game designer or content creator defines its behavior in a data file. The server's interaction system then loads and executes it.

**Hypothetical Configuration (e.g., in HOCON or JSON):**
```json
// In an item or ability definition file
{
  "onUse": {
    "type": "ConditionInteraction",
    "RequiredGameMode": "Creative",
    "Flying": true,
    "Crouching": false
  }
}
```
In this example, the interaction would only succeed if the player is in Creative mode, is flying, and is not crouching.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ConditionInteraction()`. This bypasses the codec system and results in an object with null fields for all conditions, rendering it non-functional and likely to cause `NullPointerException`s downstream.

- **Manual Invocation:** Do not call the `tick0` method directly. It must be invoked by the server's Interaction Module, which is responsible for creating and populating the `InteractionContext` with valid, tick-consistent entity data. Calling it manually will result in unpredictable behavior.

## Data Pipeline
The flow of data for this class occurs in two distinct phases: configuration and execution.

**Configuration-Time Flow:**
> Game Asset File (JSON/HOCON) → Hytale Codec System → **ConditionInteraction Instance** (In Memory)

**Runtime Execution Flow:**
> Player Input → Network Packet → Server Interaction Module → **ConditionInteraction.tick0(context)** → `InteractionContext` State Update (Finished/Failed) → Downstream Interaction Logic

