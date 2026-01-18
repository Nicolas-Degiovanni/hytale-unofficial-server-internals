---
description: Architectural reference for BuilderInstruction
---

# BuilderInstruction

**Package:** com.hypixel.hytale.server.npc.instructions.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderInstruction extends BuilderBase<Instruction> {
```

## Architecture & Concepts
The BuilderInstruction class is a foundational component of the server's NPC asset loading pipeline. It embodies the Builder pattern to translate declarative JSON configuration into a concrete, executable Instruction object. This class acts as the primary deserializer and factory for individual nodes within an NPC's behavior tree.

Architecturally, it is a composite builder. A single BuilderInstruction does not just configure an Instruction; it orchestrates the construction of all its constituent parts by delegating to other specialized builders and helpers:
*   **BuilderObjectReferenceHelper:** Manages the deferred building of singular, complex child objects like Sensor, BodyMotion, HeadMotion, and ActionList.
*   **BuilderObjectListHelper:** Manages the building of a list of nested child Instructions, enabling the creation of hierarchical and recursive behavior trees.
*   **BuilderValidationHelper:** Encapsulates complex validation logic, ensuring that the relationships and constraints between different parts of the configuration are respected at load time.

This delegation to helper objects is a key design choice, promoting separation of concerns and keeping the BuilderInstruction class focused on the high-level structure of an instruction. The system supports dynamic property resolution at build time via the ExecutionContext, allowing parts of the configuration to be evaluated rather than being purely static values.

## Lifecycle & Ownership
- **Creation:** A BuilderInstruction is never instantiated directly by game logic. It is created by the central BuilderManager service during the recursive descent of an NPC's JSON asset file. For each JSON object representing an instruction, a new BuilderInstruction instance is created to process it.

- **Scope:** The object's lifetime is strictly limited to the asset parsing and building phase. It is a short-lived, transient object that holds temporary state read from the configuration file.

- **Destruction:** Once the build method is called and the final Instruction object is returned (or determined to be null), the BuilderInstruction has fulfilled its purpose. It holds no further references and is subsequently reclaimed by the Java garbage collector.

## Internal State & Concurrency
- **State:** The internal state of a BuilderInstruction is highly **mutable**. The readConfig method populates numerous fields, effectively transforming the builder into a temporary container for the deserialized JSON data. This state is discarded after the final Instruction product is built.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The entire NPC asset loading process is designed to be a single-threaded operation. Concurrent calls to readConfig or build would result in a corrupted internal state and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | Builder<Instruction> | O(N) | Parses the input JSON, populating the builder's internal state. N is the number of keys and nested objects. |
| build(BuilderSupport support) | Instruction | O(M) | Constructs and returns the final Instruction object from the internal state. M is the number of child components. Returns null if the instruction is disabled. |
| validate(...) | boolean | O(M) | Performs load-time validation of the configuration and all its children. |
| category() | Class<Instruction> | O(1) | Returns the class literal for the object type this builder produces. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly. It is used exclusively by the NPC asset loading framework. The conceptual flow within the framework is as follows.

```java
// Conceptual example of framework usage
BuilderManager manager = ...;
JsonElement instructionJson = ...;
BuilderSupport support = ...;

// The manager instantiates and uses the builder internally
Instruction instruction = manager.build(Instruction.class, instructionJson, support);

if (instruction != null) {
    // Use the fully constructed, immutable Instruction object
    npc.getBehaviorTree().addInstruction(instruction);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new BuilderInstruction()`. The lifecycle is managed entirely by the asset loading framework, which injects necessary dependencies like resolvers and state helpers.

- **State Re-use:** Do not attempt to reuse a BuilderInstruction instance to parse a second JSON object. The builder is stateful and designed for a single, one-shot build operation.

- **Premature Building:** Do not call build before readConfig has been invoked. This will result in an incomplete or invalid Instruction object.

- **Concurrent Access:** Never access a BuilderInstruction instance from more than one thread. The class is fundamentally single-threaded in its design.

## Data Pipeline
The BuilderInstruction is a critical stage in the data transformation pipeline that turns static configuration files into live server objects.

> Flow:
> NPC Asset File (JSON) -> Framework Deserializer -> **BuilderInstruction.readConfig** -> Internal State (Helpers, Holders) -> **BuilderInstruction.build** -> Immutable Instruction Object -> NPC Behavior Engine

