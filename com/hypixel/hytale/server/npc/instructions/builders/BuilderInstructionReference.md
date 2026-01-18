---
description: Architectural reference for BuilderInstructionReference
---

# BuilderInstructionReference

**Package:** com.hypixel.hytale.server.npc.instructions.builders
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderInstructionReference extends BuilderInstruction {
```

## Architecture & Concepts
The BuilderInstructionReference is a specialized factory class within the server-side NPC behavior system. Its primary architectural function is to enable the creation of named, reusable blocks of instructions. This is a cornerstone of creating modular and maintainable NPC AI assets.

Unlike other BuilderInstruction subclasses that define immediate, inline behaviors, a BuilderInstructionReference defines a template that is not part of the default execution flow. Instead, it registers itself with a central resolver service during the asset loading phase. Other instructions, such as a GoTo instruction, can then execute this block by referencing its unique name.

The core mechanism enabling this behavior is the `excludeFromRegularBuilding` method, which signals to the asset parser that this block should be skipped during the main, sequential build pass. It is only processed and converted into a runtime Instruction object when explicitly requested by another part of the AI graph. This pattern prevents redundant processing and supports a "define once, use many times" design philosophy for complex AI behaviors.

## Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the asset deserialization engine (e.g., Gson) when a corresponding type identifier is found within an NPC behavior JSON file. It is never created manually by developers.
-   **Scope:** The object's lifetime is strictly confined to the asset loading and compilation phase. It exists to parse a JSON configuration, register its compiled form, and establish dependency links. It does not persist into the active game state and is not present during runtime gameplay.
-   **Destruction:** Once the entire NPC asset has been parsed and the final, immutable Instruction objects are constructed, the BuilderInstructionReference instance is dereferenced and becomes eligible for garbage collection. Its sole purpose is fulfilled once the asset is fully loaded.

## Internal State & Concurrency
-   **State:** The object is highly mutable during its creation and the `readConfig` phase, as it populates its internal collections and configuration from the source JSON. It caches a calculated set of dependencies in the `internalDependencies` field. After the `readConfig` method completes, its state should be considered effectively immutable.

-   **Thread Safety:** **This class is not thread-safe.** The entire NPC asset loading pipeline is designed as a single-threaded process. Concurrent calls to `readConfig` or `build` will corrupt the internal state of the builder and the central reference resolver, leading to unpredictable server behavior or crashes. All interactions must occur on the main asset loading thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Instruction | O(N) | Constructs the runtime Instruction object. N is the number of nested instructions. Throws exceptions if the build context is invalid. |
| readConfig(JsonElement) | Builder<Instruction> | O(M) | Deserializes the JSON block, populates the builder's state, and registers itself with the internal reference resolver. M is the size of the JSON object. |
| excludeFromRegularBuilding() | boolean | O(1) | Critical flag that signals to the asset parser to skip this block during the main build sequence. Always returns true. |
| getInternalDependencies() | IntSet | O(1) | Returns a cached set of indices representing other reference blocks this block depends on. Returns null if no dependencies were recorded. |

## Integration Patterns

### Standard Usage
This class is used implicitly by the asset system. A game designer defines a reusable instruction block in a JSON file, which the server parses into a BuilderInstructionReference instance. Developers do not interact with this class directly.

**Conceptual JSON Example:**
```json
{
  "instructions": [
    {
      "type": "reference",
      "name": "PatrolAndScan",
      "steps": [
        // ... list of patrol instructions
      ]
    },
    {
      "type": "goTo",
      "target": "PatrolAndScan"
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new BuilderInstructionReference()`. The object's lifecycle is managed entirely by the asset deserializer. Manual creation will bypass the critical registration and dependency tracking steps.
-   **Manual Invocation:** Do not call the `build` method directly. It depends on a carefully managed context object, `BuilderSupport`, which is orchestrated by the central asset compiler. Calling it out of sequence will produce a corrupt or incomplete Instruction.
-   **State Mutation After Load:** Modifying the state of a BuilderInstructionReference after the `readConfig` method has completed is unsupported and will lead to undefined behavior.

## Data Pipeline
The flow of data for this component is strictly part of the server's boot or asset-reload sequence.

> Flow:
> NPC Behavior JSON File -> Server Asset Deserializer -> **BuilderInstructionReference.readConfig()** -> Registration in InternalReferenceResolver -> (On-demand request from another instruction) -> **BuilderInstructionReference.build()** -> Runtime Instruction Object

