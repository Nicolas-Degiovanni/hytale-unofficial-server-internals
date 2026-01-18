---
description: Architectural reference for BuilderValidationHelper
---

# BuilderValidationHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderValidationHelper {
```

## Architecture & Concepts
The BuilderValidationHelper is a transient context object, not a service. Its primary architectural role is to act as a parameter object that aggregates all necessary state and dependencies for the validation phase of the NPC asset building pipeline.

By bundling numerous components—such as the FeatureEvaluatorHelper, InternalReferenceResolver, and StateMappingHelper—into a single container, it simplifies the method signatures of various validation subroutines. It provides a consistent, read-only view of the validation context to different stages of the process.

This class does not contain any business logic itself. It is a passive data carrier, designed to be created at the start of a validation operation and discarded upon its completion. The mutable list of readErrors serves as an output parameter, accumulating validation failures as the helper object is passed through the pipeline.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by a higher-level orchestrator, such as an NPC asset factory or builder, at the beginning of a single asset's validation sequence. It is populated with helpers and collections specific to that one asset.
- **Scope:** The object's lifetime is strictly confined to the scope of the single validation method call that creates it. It is intended for short-lived, single-use purposes.
- **Destruction:** The object becomes eligible for garbage collection as soon as the root validation process completes. There are no explicit cleanup or teardown methods.

## Internal State & Concurrency
- **State:** The BuilderValidationHelper is a **shallowly immutable** container. All of its direct fields are final and cannot be reassigned after construction. However, it holds references to external objects, some of which are mutable, most notably the readErrors and evaluators lists. Consumers of this helper object can and are expected to modify the contents of these lists.

- **Thread Safety:** **This class is not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded execution within the asset building pipeline. Concurrent access to the referenced readErrors list by multiple threads would lead to race conditions and data corruption.

## API Surface
The public API consists solely of accessors for retrieving the contextual objects required for validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the identifier of the NPC asset being validated. |
| getFeatureEvaluatorHelper() | FeatureEvaluatorHelper | O(1) | Provides access to the feature validation subsystem. |
| getInternalReferenceResolver() | InternalReferenceResolver | O(1) | Provides access to the resolver for internal asset pointers. |
| getStateMappingHelper() | StateMappingHelper | O(1) | Provides access to the state machine validation subsystem. |
| getInstructionContextHelper() | InstructionContextHelper | O(1) | Provides access to the instruction and behavior validation subsystem. |
| getExtraInfo() | ExtraInfo | O(1) | Provides access to supplementary metadata from the asset source. |
| getReadErrors() | List<String> | O(1) | Returns a reference to the mutable list where validation errors are accumulated. |
| getEvaluators() | List<Evaluator<?>> | O(1) | Returns a reference to the list of decision-making evaluators for the asset. |

## Integration Patterns

### Standard Usage
The BuilderValidationHelper is never retrieved from a service locator or dependency injection container. It is constructed by an orchestrator and passed directly as an argument to a series of validation functions.

```java
// Correct usage within a hypothetical builder class
void validateNpcAsset(NpcAssetDefinition def) {
    List<String> errorList = new ArrayList<>();
    
    // 1. Orchestrator creates the helper with all necessary context
    BuilderValidationHelper validationContext = new BuilderValidationHelper(
        def.getName(),
        new FeatureEvaluatorHelper(...),
        new InternalReferenceResolver(...),
        // ... other helpers
        errorList
    );

    // 2. Helper is passed to various validation stages
    StateValidator.validate(validationContext);
    FeatureValidator.validate(validationContext);

    // 3. Results are checked from the original error list
    if (!errorList.isEmpty()) {
        throw new AssetValidationException(errorList);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not cache and reuse a BuilderValidationHelper instance for a different asset validation. Each instance is stateful and tied to a specific asset's name and error list.
- **Multi-threaded Access:** Do not pass this object to other threads. The internal error list is not thread-safe and concurrent modifications will lead to unpredictable behavior.
- **Manual Construction:** Application-level code should not construct this class. Its creation is the responsibility of the core asset building system, which guarantees that all required helper dependencies are correctly initialized.

## Data Pipeline
The BuilderValidationHelper does not transform data itself but acts as a vehicle for context and error accumulation during the validation pipeline.

> Flow:
> NPC Asset Definition -> Asset Builder -> (creates) **BuilderValidationHelper** -> Passed to [StateValidator, FeatureValidator, ReferenceValidator] -> (populates) `readErrors` list -> Asset Builder consumes `readErrors` -> Final Asset or Error Report

