---
description: Architectural reference for BuilderInstructionRandomized
---

# BuilderInstructionRandomized

**Package:** com.hypixel.hytale.server.npc.instructions.builders
**Type:** Transient (Builder Pattern)

## Definition
```java
// Signature
public class BuilderInstructionRandomized extends BuilderInstruction {
```

## Architecture & Concepts

The BuilderInstructionRandomized class is a factory component within the server-side NPC (Non-Player Character) behavior definition pipeline. It serves as a crucial bridge between static data assets—specifically, JSON configuration files—and the live, executable instruction objects that drive NPC AI at runtime.

Its primary function is to parse a JSON object that defines a "randomized" instruction group and construct an instance of InstructionRandomized. This pattern decouples the game's AI logic from its configuration, allowing designers to define complex, non-deterministic behaviors without modifying engine code.

This builder operates as part of a larger, hierarchical build process. It is invoked when the asset parser encounters a JSON block designated for this type. The builder relies on a contextual BuilderSupport object to resolve dependencies, manage state transitions (e.g., `pushCurrentStateName`), and access shared services during the build phase. The core concept it materializes is a weighted selection mechanism, enabling an NPC to choose one of several possible actions, which is fundamental for creating less predictable and more organic-feeling AI.

## Lifecycle & Ownership

-   **Creation:** An instance of BuilderInstructionRandomized is created dynamically by a parent builder or the core asset parsing system whenever a corresponding JSON configuration block is processed. It is never instantiated directly by game logic.
-   **Scope:** The object's lifetime is extremely short. It exists only for the duration of the `readConfig` and subsequent `build` method calls. Its scope is confined to the parsing and construction of a single InstructionRandomized object.
-   **Destruction:** It becomes eligible for garbage collection immediately after the `build` method returns. It holds no persistent state and is not intended to be referenced outside of the asset-loading sequence.

## Internal State & Concurrency

-   **State:** The internal state is highly **mutable**. The `readConfig` method populates fields such as `resetOnStateChange`, `executeFor`, and a list of nested instruction builders (`steps`). This state is transient, accumulated for the sole purpose of being consumed by the `build` method.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. The entire NPC asset-building process is designed to be a single-threaded, sequential operation. Concurrent access would lead to race conditions and a corrupted AI state, as the builder modifies its internal state and interacts with a non-thread-safe BuilderSupport context.

## API Surface

The public API is designed for use by the NPC asset-building framework, not for general-purpose consumption.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | InstructionRandomized | O(N) | Constructs the final InstructionRandomized object. Complexity is O(N) where N is the number of nested instructions, as it recursively invokes their build process. Returns null on failure. |
| readConfig(JsonElement) | Builder<Instruction> | O(K) | Parses the JSON data, validates fields, and populates the builder's internal state. K is the number of keys in the JSON object. Returns a self-reference for chaining. |
| getResetOnStateChange(BuilderSupport) | boolean | O(1) | Evaluates and returns the configured `ResetOnStateChange` property using the provided execution context. |
| getExecuteFor(BuilderSupport) | double[] | O(1) | Evaluates and returns the configured `ExecuteFor` time range using the provided execution context. |

## Integration Patterns

### Standard Usage

This builder is used implicitly by the server's asset loader. A developer or designer interacts with it by defining a corresponding structure within an NPC's JSON behavior file. The framework handles the instantiation and invocation.

A typical JSON configuration processed by this builder would look like this:

```json
// Example from an NPC asset file
{
  "Type": "Randomized",
  "Name": "PatrolOrIdle",
  "Sensor": { "Ref": "IsDaytime" },
  "ResetOnStateChange": true,
  "ExecuteFor": [ 2.5, 5.0 ],
  "Instructions": [
    {
      "Type": "MoveTo",
      "Weight": 5.0,
      "Target": { "Ref": "PatrolPointA" }
    },
    {
      "Type": "PlayAnimation",
      "Weight": 2.0,
      "Animation": "idle_look_around"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance via `new BuilderInstructionRandomized()`. The asset pipeline manages its lifecycle. Manual creation will result in an unconfigured, non-functional object.
-   **State Reuse:** Do not retain a reference to this builder after the `build` method has been called. It is a single-use object, and its internal state is not valid for building a second instruction.
-   **External State Modification:** Do not attempt to modify the builder's public fields or holders after `readConfig` has been called. The internal state is considered sealed before the build phase.

## Data Pipeline

The BuilderInstructionRandomized acts as a transformation stage in the data pipeline that converts static NPC assets into live game objects.

> Flow:
> NPC JSON Asset File -> GSON Parser -> `JsonElement` -> **BuilderInstructionRandomized.readConfig()** -> Internal Builder State -> **BuilderInstructionRandomized.build()** -> `InstructionRandomized` (Runtime Object) -> NPC Behavior Engine

