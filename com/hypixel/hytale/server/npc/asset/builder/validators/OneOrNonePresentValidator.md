---
description: Architectural reference for OneOrNonePresentValidator
---

# OneOrNonePresentValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class OneOrNonePresentValidator extends Validator {
```

## Architecture & Concepts
The OneOrNonePresentValidator is a specialized component within the server-side NPC asset definition framework. Its primary function is to enforce a rule of mutual exclusivity on a set of related attributes during the asset loading and building process. It ensures that for a given group of properties, a maximum of one can be defined in the source data (e.g., a JSON file).

This validator acts as a gatekeeper for data integrity, preventing logically inconsistent or ambiguous NPC configurations. For example, it can be used to ensure an NPC behavior definition does not simultaneously specify both a *patrol route* and a *fixed guard position*.

It operates in tandem with the BuilderObjectHelper, which abstracts the presence or absence of a field from its actual value. The validator's logic is stateless and focuses exclusively on counting how many of the monitored fields are present, not what their values are.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively via the static factory method `withAttributes(String...)`. This is typically done declaratively when an asset parser or builder is configured with its set of validation rules. The static `test` and `errorMessage` methods are used directly without instantiation.
-   **Scope:** An instance of OneOrNonePresentValidator is a lightweight configuration object that holds the names of the attributes to validate. Its lifetime is typically scoped to the asset loading process. The static methods are globally accessible and have no persistent state.
-   **Destruction:** Instances are transient and are garbage collected once the asset validation phase is complete. They hold no resources and require no explicit cleanup.

## Internal State & Concurrency
-   **State:** Instances are effectively immutable. The internal `attributes` array is assigned once during construction and is never modified thereafter. The class contains no static mutable state.
-   **Thread Safety:** This class is inherently thread-safe. The static methods are pure functions, operating only on their inputs. Instances are immutable and can be safely shared and used across multiple threads without synchronization. This is critical in a server environment where assets may be loaded in parallel.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(BuilderObjectHelper<?>[] objects) | boolean | O(N) | Checks if zero or one of the provided helpers contains a present value. |
| test(boolean[] readStatus) | boolean | O(N) | Checks if zero or one of the booleans in the array is true. |
| errorMessage(String[] attributes, ...) | String | O(N) | Generates a detailed, human-readable error message listing the attributes and their presence status. |
| withAttributes(String... attributes) | OneOrNonePresentValidator | O(1) | Factory method to create a configured validator instance. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by a higher-level asset building or parsing system. The system first attempts to read a set of mutually exclusive fields into BuilderObjectHelper wrappers and then uses the static `test` method to validate the outcome. If validation fails, `errorMessage` is called to generate a diagnostic message.

```java
// Within an asset loading class
BuilderObjectHelper<String> patrolRoute = asset.readOptional("patrolRoute");
BuilderObjectHelper<Vec3> guardPosition = asset.readOptional("guardPosition");

boolean isValid = OneOrNonePresentValidator.test(patrolRoute, guardPosition);

if (!isValid) {
    String[] attrs = {"patrolRoute", "guardPosition"};
    throw new AssetParseException(
        OneOrNonePresentValidator.errorMessage(attrs, new BuilderObjectHelper[]{patrolRoute, guardPosition})
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private. Do not attempt to create instances using `new`. Always use the `withAttributes` static factory method.
-   **Ignoring Return Value:** The `test` method does not throw an exception. It is a pure check that returns a boolean. The caller is responsible for handling the failure case, typically by throwing an exception that includes the output from `errorMessage`.
-   **Manual Counting:** Do not implement your own logic to count present fields. The `test` methods are optimized and handle all necessary checks correctly.

## Data Pipeline
The OneOrNonePresentValidator is a processing stage within a larger data validation pipeline for server assets.

> Flow:
> Raw Asset File (JSON) -> Deserializer -> A set of BuilderObjectHelper instances -> **OneOrNonePresentValidator.test()** -> Boolean Result -> Asset Builder (Continues on true, throws using `errorMessage` on false)

