---
description: Architectural reference for MovementConditionInteraction
---

# MovementConditionInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class MovementConditionInteraction extends SimpleInteraction {
```

## Architecture & Concepts
The MovementConditionInteraction is a specialized control-flow node within the server's Interaction system. It functions as a high-performance, data-driven **switch statement** that routes execution to different child Interactions based on the player's current movement direction.

Architecturally, this class serves as a bridge between declarative game design and the engine's imperative execution model. Game designers define instances of this class in asset files, specifying different outcomes for each movement direction (e.g., Forward, Back, ForwardLeft). The engine then uses the static CODEC to deserialize these assets into immutable configuration objects.

Its most critical role is performed during the **compilation** phase. The `compile` method translates the declarative structure into a low-level, GOTO-like sequence of operations using an OperationsBuilder. This pre-compiles the logic into a highly efficient execution path, avoiding costly lookups or conditional branching at runtime.

This interaction is client-state dependent, as indicated by `getWaitForDataFrom` returning `WaitForDataFrom.Client`. The server-side logic explicitly waits for and relies upon the `movementDirection` reported by the client to make its routing decision.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly via the `new` keyword. They are deserialized from configuration assets (e.g., JSON files) by the engine's asset loading system using the public static `CODEC` field. This typically occurs once during server startup or when a world's asset pack is loaded.
- **Scope:** Once loaded, an instance is cached and treated as an immutable singleton for its specific asset name. It persists for the entire server session.
- **Destruction:** Instances are managed by the Java garbage collector and are cleaned up during server shutdown when the relevant classloaders are unloaded. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The internal state consists of string identifiers for child interactions (e.g., `forward`, `back`, `left`). This state is populated once during deserialization and is **effectively immutable** thereafter. The class holds no mutable state related to a specific player's interaction execution.
- **Thread Safety:** This class is **thread-safe**. Because its internal state is immutable after initialization, a single cached instance can be safely used by multiple player entities across different server threads simultaneously. All mutable execution state is passed via the `InteractionContext` parameter in the `tick0` method, ensuring no cross-player state contamination.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| compile(builder) | void | O(1) | Translates the configuration into a low-level operation graph. Called once when the interaction is loaded. |
| tick0(...) | void | O(1) | Executes the logic for a specific player. Reads movement direction from the context and performs a direct jump to the pre-compiled label. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Signals to the engine that this interaction requires state from the client before it can execute. |
| needsRemoteSync() | boolean | O(1) | Signals that this interaction's configuration must be synchronized with the client. |
| generatePacket() | Interaction | O(1) | Creates the network packet representation of this interaction's configuration. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. It is designed to be configured as a data asset. A game designer would define a `MovementConditionInteraction` in a JSON file and reference it from another interaction.

**Example Asset Definition (conceptual_sprint_attack.json):**
```json
{
  "type": "MovementConditionInteraction",
  "forward": "hytale:sprint_lunge_attack",
  "forwardLeft": "hytale:sprint_slash_attack_left",
  "forwardRight": "hytale:sprint_slash_attack_right",
  "failed": "hytale:standard_attack"
}
```
In this scenario, when a player triggers an attack while sprinting, the system would execute this interaction, which would then delegate to the appropriate lunge or slash attack based on the precise movement direction. If the player is not moving, it would fall back to the `failed` interaction.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MovementConditionInteraction()`. The object will be unconfigured and will cause NullPointerExceptions. Always define it as a data asset and let the engine's `CODEC` system manage its lifecycle.
- **State Mutation:** Do not attempt to modify the public fields (`forward`, `back`, etc.) at runtime after the asset has been loaded. The compiled operation graph will not reflect these changes, leading to unpredictable behavior.

## Data Pipeline

> **Compilation Flow (at Server Load):**
> Asset File (.json) → AssetManager → `MovementConditionInteraction.CODEC` → **MovementConditionInteraction Instance** → `compile()` → In-Memory Operation List

> **Execution Flow (during Gameplay):**
> Client Input (e.g., W + Left-Click) → ClientState Network Packet → Server Network Layer → `InteractionContext` populated with `movementDirection` → `tick0()` on **MovementConditionInteraction** → `context.jump()` → Execution of pre-compiled child Interaction (e.g., `hytale:sprint_slash_attack_left`)

