---
description: Architectural reference for StringsOneSetValidator
---

# StringsOneSetValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class StringsOneSetValidator extends Validator {
```

## Architecture & Concepts
The StringsOneSetValidator is a specialized, stateless utility class designed to enforce a rule of mutual exclusivity between two string-based attributes within the server's NPC asset definition pipeline. Its primary function is to ensure data integrity by guaranteeing that for a given pair of properties, exactly one is defined with a non-empty value.

This component is a critical part of the asset validation stage, which runs before an NPC asset is fully loaded and instantiated in the game world. It prevents logical contradictions in NPC behavior definitions, such as an NPC being configured to have both a fixed spawn point and a dynamic patrol route simultaneously. By failing fast during the server's boot or asset-loading sequence, it prevents difficult-to-diagnose runtime errors caused by ambiguous configurations.

## Lifecycle & Ownership
- **Creation:** As a utility class, its static methods are available for the lifetime of the JVM. Instances, which are primarily data-holders for attribute names, are created on-demand via the static factory methods `withAttributes`. These instances are typically instantiated by a higher-level asset builder or a generic validation framework that requires a `Validator` object.
- **Scope:** The scope of any given instance is transient and typically confined to a single validation operation. They are created, used to generate a potential error message, and then immediately become eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The class is effectively stateless. The static methods are pure functions that operate solely on their inputs. Instances created via the factory methods hold a final `attributes` array, making them immutable after construction.
- **Thread Safety:** This class is inherently thread-safe. Its stateless nature and the immutability of its instances allow it to be safely used in multi-threaded asset loading and validation pipelines without requiring any external locking or synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String, String) | static boolean | O(1) | The core validation logic. Returns true if exactly one of the two provided strings is non-null and non-empty. |
| errorMessage(...) | static String | O(1) | Generates a standardized, user-friendly error message for logging or build failure reports. |
| withAttributes(...) | static StringsOneSetValidator | O(1) | Factory method to create an immutable validator instance, typically for integration with a generic validation system. |

## Integration Patterns

### Standard Usage
This validator is most commonly invoked directly by an asset parsing service to perform an immediate check on two conflicting fields.

```java
// In an asset loading class
String patrolRoute = npcDefinition.getPatrolRoute();
String fixedPosition = npcDefinition.getFixedPosition();

if (!StringsOneSetValidator.test(patrolRoute, fixedPosition)) {
    throw new AssetBuildException(
        StringsOneSetValidator.errorMessage("patrolRoute", "fixedPosition", npcDefinition.getId())
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using `new StringsOneSetValidator()`. Always use the static `withAttributes` factory methods if an object instance is required.
- **Ignoring Empty Strings:** The validation logic correctly treats a null string and an empty string as equivalent. Do not perform a separate null check before calling `test`, as this can lead to incorrect validation outcomes. The method is designed to handle both cases.

## Data Pipeline
The validator operates as a gate within the broader NPC asset data pipeline. It does not transform data but rather asserts its structural correctness.

> Flow:
> NPC Asset File (JSON/YAML) -> Deserializer -> Raw Data Object -> **StringsOneSetValidator.test()** -> [If False: Halt & Error] -> [If True: Continue Build] -> Live NPC Object

