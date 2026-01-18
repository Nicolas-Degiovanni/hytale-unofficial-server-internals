---
description: Architectural reference for OnePresentValidator
---

# OnePresentValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class OnePresentValidator extends Validator {
```

## Architecture & Concepts
The OnePresentValidator is a specialized component within the server's NPC asset construction framework. Its primary architectural role is to enforce a mutual exclusivity constraint on a set of related attributes during the asset validation phase. This is critical for asset definitions where multiple, competing properties could be defined, but only one is semantically valid at a time (e.g., an entity can have a *model* or a *mesh*, but not both).

This class serves a dual purpose:
1.  **Static Utility:** It provides a suite of static, pure functions for performing immediate, ad-hoc validation checks. This allows developers to directly invoke the validation logic without needing to instantiate an object.
2.  **Validator Factory:** Through the `withAttributes` factory method, it can produce configured instances of itself. These instances are designed to be integrated into a larger, declarative validation pipeline where a collection of `Validator` objects are executed against an asset builder.

The core logic revolves around the `BuilderObjectHelper`, an abstraction that represents a potentially present attribute read from a source configuration file. The validator inspects an array of these helpers to ensure that the count of "present" helpers is exactly one.

## Lifecycle & Ownership
The OnePresentValidator has no complex lifecycle, as it is fundamentally a stateless utility.

*   **Creation:** As a utility class, its static methods are available at all times after class loading. When used as an instance, it is created exclusively via the static factory method `withAttributes`. This is typically done once during the setup of an asset builder's validation rules.
*   **Scope:** The static methods have a global scope. Instances are transient and short-lived, typically scoped to a single asset build or validation operation. They hold no external resources or references that would persist beyond this scope.
*   **Destruction:** Instances are lightweight and are reclaimed by the garbage collector once the validation process that created them is complete. No explicit cleanup is required.

## Internal State & Concurrency
*   **State:** The class is effectively stateless. Its static methods are pure functions that depend only on their arguments. Instances created via the factory method are **immutable**; the internal `attributes` array is final and is never mutated after construction.
*   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable and stateless design, all methods can be called concurrently from multiple threads without any risk of race conditions or data corruption. No locks or synchronization primitives are necessary. This makes it highly suitable for use in parallel asset processing systems.

## API Surface
The public API consists entirely of static methods for performing validation and generating descriptive error messages.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(String... attributes) | OnePresentValidator | O(1) | Factory method to create a configured validator instance. |
| test(BuilderObjectHelper<?>[] objects) | boolean | O(N) | The primary validation function. Returns true if exactly one helper is present. |
| test(boolean[] readStatus) | boolean | O(N) | An alternative validation function that operates on a pre-computed array of presence flags. |
| countPresent(int size, IntPredicate p) | int | O(N) | The core counting logic. Iterates N elements and counts how many satisfy the predicate. |
| errorMessage(String[] attrs, ...) | String | O(N) | Constructs a detailed, human-readable error message for validation failures. |

## Integration Patterns

### Standard Usage
The most common pattern is to use the static `test` and `errorMessage` methods together to perform a validation check and throw a descriptive exception upon failure.

```java
// In a hypothetical asset builder:
BuilderObjectHelper<Model> model = ...;
BuilderObjectHelper<Mesh> mesh = ...;
BuilderObjectHelper<Prefab> prefab = ...;

BuilderObjectHelper<?>[] options = {model, mesh, prefab};
String[] attributeNames = {"model", "mesh", "prefab"};

if (!OnePresentValidator.test(options)) {
    String error = OnePresentValidator.errorMessage(attributeNames, options);
    throw new AssetValidationException(error);
}
```

### Anti-Patterns (Do NOT do this)
*   **Direct Instantiation:** The constructor is private and inaccessible. **Warning:** Do not attempt to use reflection to create instances. Always use the `withAttributes` static factory method.
*   **Misinterpreting Counts:** Avoid using `countPresent` for validation logic. The `test` method correctly implements the `count == 1` check. Using `countPresent` directly can lead to incorrect validation logic, for example, by checking `countPresent() > 0` which allows multiple attributes to be present.
*   **Mismatched Attribute Names:** When calling `errorMessage`, the `String[] attributes` array must have the same length and correspond positionally to the `BuilderObjectHelper<?>[] objects` array. A mismatch will produce a confusing and misleading error message for the user.

## Data Pipeline
This component acts as a gate in a data validation pipeline, not a transformation stage. It consumes state information and produces a binary outcome.

> Flow:
> Asset Configuration (e.g., JSON) -> Deserializer -> Array of `BuilderObjectHelper` -> **OnePresentValidator.test()** -> Validation Result (Boolean) -> (On Failure) -> **OnePresentValidator.errorMessage()** -> Exception Thrown

