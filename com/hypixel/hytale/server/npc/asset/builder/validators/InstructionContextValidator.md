---
description: Architectural reference for InstructionContextValidator
---

# InstructionContextValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient

## Definition
```java
// Signature
public class InstructionContextValidator extends Validator {
```

## Architecture & Concepts
The InstructionContextValidator is a specialized rule-enforcement component within the server-side NPC Asset Builder framework. Its primary architectural role is to act as a contextual predicate, ensuring that specific asset properties or instructions are only declared within their designated, valid scopes.

This validator is not a general-purpose tool; it is purpose-built to understand the nested structure of NPC asset definitions. It operates by being configured with a set of allowed *Instruction Types* (e.g., `add_component`, `set_property`) and a set of allowed *Component Contexts* (e.g., `on_spawn`, `on_death`). During the asset build process, the validator checks the current parsing context against its configured allow-lists. If the current context is not permitted, the build process is halted with a precise error.

This mechanism is critical for maintaining asset integrity and preventing logical errors where a developer might, for example, attempt to define a spawn-related property within a death-event block.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively via the static factory method `inInstructions`. The constructor is private to prevent direct instantiation, enforcing a clear and controlled creation pattern. It is typically instantiated by higher-level asset definition systems when building a validation chain for a specific property.
- **Scope:** Short-lived and transient. An instance of InstructionContextValidator exists only for the duration of a single validation pass over a portion of an NPC asset. It is not registered, cached, or shared across different asset builds.
- **Destruction:** The object is immediately eligible for garbage collection once the validation check is complete. It holds no external resources or references and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** Immutable. The internal state, consisting of two EnumSets for instruction types and component contexts, is set once at creation time and cannot be modified thereafter. All fields are final.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. A single instance can be safely used across multiple threads without synchronization. However, the standard usage pattern involves creating a new instance for each validation task within a single-threaded asset build pipeline.

## API Surface
The public API is minimal and focused on creation and error reporting. The core validation logic is invoked through the parent `Validator` interface, which is not detailed here.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| inInstructions(instructionTypes, componentContexts) | InstructionContextValidator | O(N+M) | Static factory method. Creates and returns a new validator configured with the specified sets of allowed contexts. |
| getErrorMessage(...) | String | O(1) | Static utility method. Constructs a detailed, human-readable error message for reporting validation failures. |

## Integration Patterns

### Standard Usage
This validator is intended to be used declaratively within the asset building framework. A developer defines a validation rule by calling the factory method and attaching the resulting validator to an asset property definition.

```java
// Example from a hypothetical AssetPropertyBuilder
// This rule ensures 'my_property' is only used within 'add_component'
// instructions and during the 'on_spawn' or 'on_tick' contexts.

AssetPropertyBuilder.create("my_property")
    .withValidator(
        InstructionContextValidator.inInstructions(
            EnumSet.of(InstructionType.ADD_COMPONENT),
            EnumSet.of(ComponentContext.ON_SPAWN, ComponentContext.ON_TICK)
        )
    );
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance using `new InstructionContextValidator()`. The constructor is private by design. Always use the `inInstructions` static factory method.
- **Stateful Reuse:** Do not hold onto an instance of this validator for reuse in different validation scenarios. Its configuration is immutable. Create a new, correctly configured instance for each distinct validation rule.
- **Manual Error Formatting:** Do not call `getErrorMessage` directly in application logic. This is a low-level helper intended for use by the core validation engine, which supplies the necessary context parameters.

## Data Pipeline
The InstructionContextValidator functions as a gate within the larger NPC asset data processing pipeline. It does not transform data but rather asserts the validity of the context in which data appears.

> Flow:
> NPC Asset File (JSON) -> Jackson Parser -> Asset Builder -> **InstructionContextValidator** -> Validation Result (Success or Error) -> Compiled NPC Asset<ctrl63>

