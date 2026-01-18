---
description: Architectural reference for FeatureEvaluatorHelper
---

# FeatureEvaluatorHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class FeatureEvaluatorHelper {
```

## Architecture & Concepts
The FeatureEvaluatorHelper is a stateful, short-lived container object used exclusively during the server-side NPC asset compilation pipeline. Its primary role is to aggregate evaluation logic and validation rules for a specific NPC "feature" as it is being parsed from a definition file.

This class embodies the **Builder** pattern for a collection of rules. It operates in two distinct phases:

1.  **Configuration Phase:** An asset parser or builder instantiates a FeatureEvaluatorHelper and populates it with ProviderEvaluator instances and validation callbacks (BiConsumers). During this phase, the internal collections are mutable.
2.  **Execution Phase:** After all rules have been added, the `lock()` method is called. This transitions the helper into a read-only state, making its list of evaluators immutable. The owning builder can then safely retrieve the collected evaluators and execute the registered validation routines.

This mechanism decouples the *collection* of rules from their *validation and use*, allowing for a multi-stage asset compilation process where all components can be parsed first and then validated in a second pass.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via `new FeatureEvaluatorHelper()` by a higher-level builder or parser responsible for interpreting a single, discrete part of an NPC asset definition.
-   **Scope:** The object's lifetime is confined to the scope of a single asset build operation. It is created, populated, locked, used for validation, and then immediately becomes eligible for garbage collection. It does not persist beyond the asset compilation process.
-   **Destruction:** Managed by the Java garbage collector. There are no manual cleanup steps required. Once the builder that created it completes its task, the helper is no longer referenced and will be reclaimed.

## Internal State & Concurrency
-   **State:** Highly mutable before the `lock()` method is invoked. Its internal lists of evaluators and validators are populated dynamically. After `lock()` is called, the primary list of ProviderEvaluators becomes immutable. The boolean flags remain technically mutable but are not intended to be changed after the initial configuration.
-   **Thread Safety:** **This class is not thread-safe and must be thread-confined.** It is designed for use within a single-threaded asset parsing and compilation loop. It contains no internal synchronization mechanisms. Concurrent access will lead to race conditions, `ConcurrentModificationException`, and an indeterminate internal state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(ProviderEvaluator) | void | O(1) | Adds an evaluator to the internal list. Throws `UnsupportedOperationException` if called after `lock()`. |
| lock() | FeatureEvaluatorHelper | O(1) | Finalizes the helper, making the internal provider list immutable. This is a terminal operation. |
| addProviderReferenceValidator(...) | void | O(1) | Registers a validation callback for provider references. |
| addComponentRequirementValidator(...) | void | O(1) | Registers a validation callback for component-level requirements. |
| validateProviderReferences(...) | void | O(N) | Executes all registered provider reference validators. N is the number of validators. |
| validateComponentRequirements(...) | void | O(M) | Executes all registered component requirement validators. M is the number of validators. |
| getProviders() | List<ProviderEvaluator> | O(1) | Returns the list of collected evaluators. The list is unmodifiable if `lock()` has been called. |

## Integration Patterns

### Standard Usage
The intended use is a sequential build-lock-validate pattern, typically orchestrated by a larger asset builder.

```java
// 1. A builder creates a fresh helper for a new component
FeatureEvaluatorHelper helper = new FeatureEvaluatorHelper();

// 2. The parser populates it with evaluators and validators
//    based on the asset definition file.
helper.add(new SomeProviderEvaluator());
helper.addComponentRequirementValidator((h, context) -> {
    if (!h.belongsToFeatureRequiringComponent()) {
        context.addError("This component is missing a required feature flag.");
    }
});

// 3. Once parsing for this component is complete, it is locked.
helper.lock();

// 4. The main builder can now safely execute validation passes.
ExecutionContext context = new ExecutionContext();
helper.validateComponentRequirements(helper, context);

// 5. The finalized providers are retrieved for runtime use.
List<ProviderEvaluator> evaluators = helper.getProviders();
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Do not re-use a FeatureEvaluatorHelper instance for building multiple, independent components. Its internal state is not designed to be reset. Create a new instance for each distinct build task.
-   **Modification After Lock:** Do not attempt to call `add()` after `lock()` has been invoked. This will result in an `UnsupportedOperationException` and indicates a flaw in the build process logic.
-   **Skipping Validation:** The validation methods are a critical part of the asset pipeline. Retrieving providers without first running the validation steps can lead to subtle and difficult-to-diagnose runtime errors.

## Data Pipeline
FeatureEvaluatorHelper acts as a temporary staging area during the configuration of the NPC asset pipeline. It does not process data at runtime.

> Configuration Flow:
> NPC Asset File (JSON/HOCON) -> Asset Parser -> **FeatureEvaluatorHelper.add()** -> **FeatureEvaluatorHelper.lock()** -> Main Builder consumes `getProviders()` -> **FeatureEvaluatorHelper.validate...()** -> Finalized NPC Component<ctrl63>

