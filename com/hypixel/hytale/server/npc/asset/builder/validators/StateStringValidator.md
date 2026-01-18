---
description: Architectural reference for StateStringValidator
---

# StateStringValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient

## Definition
```java
// Signature
public class StateStringValidator extends StringValidator {
```

## Architecture & Concepts
The StateStringValidator is a specialized utility component responsible for parsing and validating strings that represent entity or component states. These states are expected to follow a `mainState.subState` convention, a common pattern in the engine for defining hierarchical states in configuration files, such as those for NPC behaviors or block properties.

This class acts as a gatekeeper during the asset loading and deserialization pipeline. By encapsulating the specific rules of state string formatting, it decouples the asset builders from the raw string manipulation logic. Its design employs a set of static factory methods (e.g., `mainStateOnly`, `requireMainState`) to provide pre-configured validator instances, which simplifies its use and promotes consistent validation rules across the server codebase.

A key architectural choice is that the `test` method not only returns a boolean result but also mutates the validator's internal state by caching the parsed state components. This allows a consumer to validate and extract the components in a single, efficient operation without needing to re-parse the string.

## Lifecycle & Ownership
-   **Creation:** Instances are never created directly via a constructor. They are exclusively provisioned through one of the static factory methods: `get`, `mainStateOnly`, `requireMainState`, or `requireMainStateOrNull`. This class is not managed by a dependency injection framework.

-   **Scope:** Transient and short-lived. An instance of StateStringValidator is intended to be used for a single, immediate validation operation. Its internal state, specifically the parsed `stateParts`, is only considered valid in the moments immediately following a successful call to the `test` method.

-   **Destruction:** The object is lightweight and holds no external resources. It becomes eligible for garbage collection as soon as it falls out of the scope of the method that created it. There is no explicit destruction or cleanup required.

## Internal State & Concurrency
-   **State:** This class is stateful and highly mutable. The configuration flags (`allowEmptyMain`, `mainStateOnly`, `allowNull`) are immutable and set at creation time. However, the `stateParts` string array is a mutable field that is overwritten as a side effect of every call to the `test` method.

-   **Thread Safety:** **This class is not thread-safe.** Sharing a single instance of StateStringValidator across multiple threads will lead to severe race conditions. If one thread calls `test`, it will overwrite the `stateParts` field, potentially corrupting the results for another thread that has just completed its own `test` call and is about to call `getMainState` or `getSubState`.

    **WARNING:** Always create a new validator instance for each distinct validation operation. Do not store instances in shared static fields or pass them between threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static StateStringValidator | O(1) | Factory method. Creates a permissive validator. |
| mainStateOnly() | static StateStringValidator | O(1) | Factory method. Creates a validator that disallows sub-states. |
| requireMainState() | static StateStringValidator | O(1) | Factory method. Creates a strict validator requiring a non-empty main state. |
| requireMainStateOrNull() | static StateStringValidator | O(1) | Factory method. Creates a strict validator that also permits null input. |
| test(String value) | boolean | O(N) | Validates the input string. **WARNING:** This method mutates the internal state of the object. |
| getMainState() | String | O(1) | Returns the main state component. Throws if called before a successful `test`. |
| getSubState() | String | O(1) | Returns the sub-state component. Throws if called before a successful `test`. |
| hasMainState() | boolean | O(1) | Checks for the presence of a main state. Only reliable after `test` is called. |
| hasSubState() | boolean | O(1) | Checks for the presence of a sub-state. Only reliable after `test` is called. |

## Integration Patterns

### Standard Usage
The correct pattern is to create a validator, immediately call `test`, and then, within the scope of a successful result, extract the parsed components.

```java
// How a developer should normally use this
String inputState = "walking.patrol";
StateStringValidator validator = StateStringValidator.requireMainState();

if (validator.test(inputState)) {
    // Safe to access state components here
    String main = validator.getMainState(); // "walking"
    String sub = validator.getSubState();   // "patrol"
    // ... proceed with game logic
} else {
    // Handle invalid state string
    log.warn(validator.errorMessage(inputState));
}
```

### Anti-Patterns (Do NOT do this)
-   **State Access Before Validation:** Accessing state-retrieval methods like `getMainState` before `test` has been called and returned true will result in an `ArrayIndexOutOfBoundsException` or return stale data from a previous operation.

-   **Instance Reuse:** Reusing the same validator instance for multiple, unrelated validations is dangerous due to its mutable state. The results of one `test` call will be overwritten by the next.

-   **Concurrent Access:** As detailed under Concurrency, sharing an instance across threads is strictly forbidden and will lead to unpredictable behavior and data corruption.

## Data Pipeline
This validator typically sits within a data deserialization or configuration pipeline, acting as a transformation and validation step.

> Flow:
> NPC Asset File (JSON) -> Jackson Deserializer -> **StateStringValidator** -> NPC State Machine Configuration

