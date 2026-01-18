---
description: Architectural reference for SensorEval
---

# SensorEval

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Component

## Definition
```java
// Signature
public class SensorEval extends SensorBase {
```

## Architecture & Concepts
SensorEval is a specialized component within the Hytale NPC AI framework, designed to evaluate custom expressions as a condition for AI behavior. It functions as a dynamic, data-driven predicate, allowing game designers to define complex logical conditions in configuration files rather than hard-coding them in Java.

The core architectural principle is **Compile-Once, Execute-Many**. During the NPC's initialization phase, a human-readable expression string is parsed and compiled into an efficient, low-level instruction set. This pre-compiled bytecode is then cached within the SensorEval instance.

At runtime, the `matches` method is invoked repeatedly by the AI system. Instead of re-parsing the string, it executes the cached instruction set against the current game state, providing significant performance gains. This component effectively acts as a bridge between the server's declarative asset system and the imperative logic of the AI update loop.

## Lifecycle & Ownership
- **Creation:** An instance of SensorEval is created by the server's asset loading pipeline when an NPC's AI definition is being constructed. It is not instantiated directly but through a corresponding builder, `BuilderSensorEval`, which provides the necessary expression string and build context.
- **Scope:** The object's lifetime is tightly coupled to its parent NPC entity. It persists as a component of the NPC's AI logic for as long as the NPC is active in the world.
- **Destruction:** The instance is eligible for garbage collection when the owning NPC entity is unloaded or destroyed. There are no explicit cleanup methods.

## Internal State & Concurrency
- **State:** SensorEval is stateful, but its state is effectively **immutable after construction**. The expression string, compiled instructions, and validity flag are all finalized within the constructor. This immutability ensures predictable behavior during runtime execution.

- **Thread Safety:** This class is **not thread-safe**. The underlying `ExecutionContext` used for evaluating expressions maintains an internal stack and is not designed for concurrent access. The entire NPC AI update cycle, including sensor evaluation, is expected to operate within a single thread for a given world or simulation region.

    **WARNING:** Invoking `matches` from multiple threads on the same instance will result in race conditions and catastrophic state corruption.

## API Surface
The public contract is minimal, exposing only the essential behavior inherited from its base class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Executes the pre-compiled expression. N is the number of instructions. Returns true if the expression evaluates to true and the sensor is valid. |
| getSensorInfo() | InfoProvider | O(1) | Not implemented. Returns null. |

## Integration Patterns

### Standard Usage
SensorEval is intended to be defined declaratively within an NPC's asset files. The system handles its instantiation and integration into the AI's behavior tree or state machine. The AI engine then calls `matches` during its update tick to query the condition.

```java
// This class is not used directly in code by developers.
// It is configured in a data file (e.g., JSON or HOCON)
// and instantiated by the engine.

/* Example pseudo-asset definition:
  behavior: {
    type: "state_machine",
    states: [
      {
        name: "Fleeing",
        enter_conditions: [
          {
            type: "SensorEval",
            expression: "self.health < 0.25 && target.distance < 10"
          }
        ]
      }
    ]
  }
*/
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorEval()`. The constructor requires a `BuilderSensorEval` and a `BuilderSupport` context, which are only available during the server's asset loading process. Bypassing this will result in a component that cannot function.
- **Runtime Expression Changes:** The expression is compiled once at creation. Do not attempt to modify the expression string at runtime via reflection or other means; the compiled instructions will not be updated.
- **Ignoring Compilation Errors:** The constructor will throw a `RuntimeException` if the expression is syntactically incorrect or does not evaluate to a boolean. This is a fatal error by design, indicating a problem with the asset configuration. Systems should not catch this exception and allow the NPC to spawn in a broken state.

## Data Pipeline
The component transforms a declarative string from a configuration file into an executable boolean result at runtime.

> Flow:
> NPC Asset File -> Asset Deserializer -> `BuilderSensorEval` -> **`SensorEval` Constructor (Compilation)** -> `Instruction[]` Cache -> AI Update Loop -> `matches()` Call -> **`SensorEval` (Execution)** -> Boolean Result -> AI State Transition

