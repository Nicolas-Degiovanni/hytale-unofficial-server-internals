---
description: Architectural reference for InstructionContextHelper
---

# InstructionContextHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class InstructionContextHelper {
```

## Architecture & Concepts
The InstructionContextHelper is a stateful, transient object that serves as a context and validation manager within the NPC asset building pipeline. Its primary role is to track the state of an instruction parser as it processes different sections of an asset definition, particularly distinguishing between general instructions and those specific to a component.

This class is not a standalone service but rather a tightly-coupled helper to a larger orchestrator, such as an asset builder or parser. It encapsulates the logic for "what is currently being built" (the context) and "are the current operations valid for this context" (the validation).

The core design revolves around two levels of context:
1.  **InstructionType:** The primary context, defining the broad category of the current operation (e.g., Component, Metadata). This is immutable after creation.
2.  **ComponentContext:** A secondary, more granular context that is only relevant when the InstructionType is Component (e.g., Appearance, Physics).

A key architectural feature is its dynamic validation system using a list of evaluators (BiConsumers). This allows the primary builder logic to register context-sensitive validation rules on the fly. These rules are then executed by the helper at appropriate checkpoints, decoupling the validation logic from the main parsing loop.

## Lifecycle & Ownership
-   **Creation:** An instance is created by a parent builder or parser at the start of a specific, bounded task, such as parsing a single asset file. The initial InstructionType is provided upon construction.
-   **Scope:** The object's lifetime is strictly limited to the duration of the single asset-building operation it was created for. It is designed to be short-lived.
-   **Destruction:** The object holds no external resources and does not require explicit cleanup. It is eligible for garbage collection as soon as the parent builder or parser that created it completes its task and the reference is dropped.

## Internal State & Concurrency
-   **State:** This class is highly mutable and stateful by design. The initial *context* field is final, but the *componentContext* can be changed, and the list of *componentContextEvaluators* is lazily initialized and appended to during its lifecycle.
-   **Thread Safety:** **This class is not thread-safe.** All fields are accessed without any synchronization mechanisms. The lazy initialization of the evaluators list and subsequent additions are not atomic. It is critically important that a single instance is confined to the single thread performing the asset build operation.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| InstructionContextHelper(InstructionType) | constructor | O(1) | Creates a new helper with a fixed primary context. |
| setComponentContext(ComponentContext) | void | O(1) | Sets the secondary, more granular context. |
| isInCorrectInstruction(EnumSet) | boolean | O(1) | Checks if the primary context is within a set of valid types. |
| extraContextMatches(EnumSet) | boolean | O(1) | Checks if the secondary context is within a set of valid types. |
| addComponentContextEvaluator(BiConsumer) | void | O(1) | Registers a new validation rule to be executed later. |
| validateComponentContext(...) | void | O(N) | Executes all registered validation rules. Throws IllegalStateException if the primary context is not Component. |

## Integration Patterns

### Standard Usage
The helper is instantiated by a builder, its state is updated as the asset is parsed, and its validation methods are called at key checkpoints to ensure asset integrity.

```java
// A builder creates a helper for the scope of a component block
InstructionContextHelper helper = new InstructionContextHelper(InstructionType.Component);
helper.setComponentContext(ComponentContext.Appearance);

// The builder registers a validation rule specific to this context
helper.addComponentContextEvaluator((instruction, component) -> {
    if (component == ComponentContext.Appearance && instruction == InstructionType.Physics) {
        throw new AssetParseException("Physics instructions are not allowed in an Appearance component.");
    }
});

// As the builder parses instructions, it validates them
// This would typically be inside a loop
helper.validateComponentContext(parsedInstruction, helper.getComponentContext());
```

### Anti-Patterns (Do NOT do this)
-   **Instance Reuse:** Do not reuse an InstructionContextHelper instance across multiple, independent asset build sessions. Its internal state is not reset and will cause unpredictable validation behavior.
-   **Concurrent Access:** Do not share an instance across multiple threads. This will lead to race conditions, particularly with the list of evaluators, resulting in lost validations or runtime exceptions.
-   **State Negligence:** Do not call validateComponentContext if the primary context is not Component. While the method has a guard, relying on it indicates a flaw in the calling builder's logic.

## Data Pipeline
This component does not process a data stream in a traditional sense. Instead, it validates the control flow of its parent builder.

> Flow:
> Asset Parser Begins -> **InstructionContextHelper (Created)** -> Parser Enters Component Block -> **Helper State Updated (setComponentContext)** -> Parser Registers Validation Rules -> **Helper State Updated (addComponentContextEvaluator)** -> Parser Processes Instruction -> **Helper Validates Control Flow (validateComponentContext)** -> Exception or Continued Parsing

