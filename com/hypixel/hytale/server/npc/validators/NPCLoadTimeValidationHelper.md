---
description: Architectural reference for NPCLoadTimeValidationHelper
---

# NPCLoadTimeValidationHelper

**Package:** com.hypixel.hytale.server.npc.validators
**Type:** Transient

## Definition
```java
// Signature
public class NPCLoadTimeValidationHelper {
```

## Architecture & Concepts
The NPCLoadTimeValidationHelper is a stateful, short-lived context object used exclusively during the parsing and validation of a single NPC definition file. It acts as a centralized accumulator for validation state, ensuring that all components of a complex NPC configuration are consistent with each other and with the core game assets.

Its primary role is to be passed through the recursive descent of the NPC parsing logic. Instead of passing numerous individual parameters and state collections between parsing functions, the system passes a single instance of this helper. This pattern simplifies the parsing code and centralizes all validation rules and error reporting for a single load operation.

Key responsibilities include:
-   **Asset Validation:** Verifying that referenced assets, such as animations, exist within the NPC's specified model.
-   **Component Coherency:** Ensuring that required components, like specific MotionControllers, are correctly provided by the NPC's configuration.
-   **Inventory & Slot Checks:** Validating that any equipment or item placements reference valid slots within the NPC's defined inventory dimensions.
-   **Duplicate and Circular Dependency Detection:** Using internal stacks to track context and detect redundant or illogical configurations within nested structures like behavior trees or state machines.

This class is fundamental to server stability, as it catches configuration errors at load-time, preventing runtime exceptions or undefined NPC behavior during gameplay.

## Lifecycle & Ownership
-   **Creation:** An instance is created by the NPC definition loading system at the beginning of parsing a single NPC file. The constructor is passed initial, high-level context like the file name and the primary model asset.
-   **Scope:** The object's lifetime is strictly bound to the parsing of one NPC file. It is created, used extensively throughout the parsing process, and then becomes eligible for garbage collection once the final NPC object is constructed or the parsing fails.
-   **Destruction:** There is no explicit destruction method. The object is discarded after the load operation completes, and its memory is reclaimed by the JVM garbage collector. It is critical that a new instance is created for each file; instances must not be reused.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. Its entire purpose is to accumulate state as the parser traverses an NPC definition. It maintains collections of evaluated animations, required motion controllers, and contextual stacks for tracking the current parsing state. This state is built up progressively and used to perform cross-cutting validation checks.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use during a file parsing operation. Its internal collections (HashSet, ArrayDeque) are not concurrent, and its stateful, sequential API (e.g., push/pop operations) would be corrupted by concurrent access, leading to unpredictable validation results.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setInventorySizes(int, int, int) | void | O(1) | Initializes the inventory dimensions. Must be called early in the parsing process. |
| validateAnimation(String) | void | O(1) | Checks if an animation exists on the spawn model. Caches results to avoid redundant lookups. |
| registerMotionControllerType(Class) | void | O(1) | Registers a MotionController provided by the NPC configuration. |
| requireMotionControllerType(Class) | void | O(1) | Registers a MotionController required by a component, like a behavior. |
| validateMotionControllers(List) | boolean | O(N*M) | Cross-references all required controllers against all provided ones. Populates the error list on mismatch. |
| validateInventoryHasSlot(int, String, List) | boolean | O(1) | Validates that a given slot index is within the bounds of the main inventory. |
| pushCurrentStateName(String) | void | O(1) | Pushes a state name onto the context stack for more precise error logging. |
| popCurrentStateName() | void | O(1) | Pops a state name from the context stack. |
| pushFilterSet() / popFilterSet() | void | O(1) | Manages a stack of seen filter strings to detect duplicates within a specific scope. |
| hasSeenFilter(String) | boolean | O(1) | Checks if a filter has been seen in the current scope, and adds it if not. |

## Integration Patterns

### Standard Usage
The helper is instantiated at the top level of the parsing logic for a single file. It is then passed down through subsequent calls that parse different sections of the file. Finally, aggregate validation methods are called before the NPC is considered fully loaded.

```java
// In a hypothetical NPCParser class
public ValidatedNPC parse(File npcFile) {
    List<String> errors = new ArrayList<>();
    Model npcModel = assetManager.getModel(...);

    // 1. Create a new helper for this specific file
    NPCLoadTimeValidationHelper helper = new NPCLoadTimeValidationHelper(
        npcFile.getName(), npcModel, false
    );

    // 2. Pass the helper through various parsing stages
    parseBehaviors(npcJson.get("behaviors"), helper);
    parseInventory(npcJson.get("inventory"), helper);
    parseStateMachines(npcJson.get("states"), helper);

    // 3. Run final validation checks
    helper.validateMotionControllers(errors);
    // ... other final checks

    if (!errors.isEmpty()) {
        throw new NPCParseException(errors);
    }
    return new ValidatedNPC(...);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Re-use:** Never re-use a helper instance to validate a second NPC file. The internal state, such as `evaluatedAnimations`, will carry over and produce incorrect validation results. Always create a new instance for each file.
-   **Concurrent Modification:** Do not access or modify a helper instance from multiple threads. The parsing process for a single file must be confined to a single thread.
-   **Unbalanced Stack Operations:** Failing to call `popCurrentStateName` or `popFilterSet` after a corresponding `push` operation, especially in the case of an exception, will corrupt the helper's state for all subsequent parsing within the same file. Use `try...finally` blocks to ensure pops are always executed.

## Data Pipeline
This class acts as a validation sink rather than a data transformer. It consumes metadata from the parser and produces a list of validation errors.

> Flow:
> NPC Definition File -> JSON Parser -> **NPCLoadTimeValidationHelper** (State Accumulated) -> Final Validation -> List of Errors OR Success


