---
description: Architectural reference for ExistsIfParameterSetValidator
---

# ExistsIfParameterSetValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Rule Object

## Definition
```java
// Signature
public class ExistsIfParameterSetValidator extends Validator {
```

## Architecture & Concepts
The ExistsIfParameterSetValidator is a concrete implementation of the **Specification Pattern**, designed to enforce a single, specific rule within the server's NPC asset validation framework. It represents a conditional dependency: if a designated *parameter* is defined in an asset, then a corresponding *attribute* must also be present.

This class is not a standalone service but rather a small, immutable data object that encapsulates a piece of validation logic. It is designed to be aggregated by a higher-level validation engine or asset builder. This engine iterates through a collection of Validator objects, including instances of this class, to ensure the integrity and correctness of NPC asset definitions before they are loaded into the game world. Its existence decouples the complex validation logic from the core asset parsing and building process, promoting modularity and maintainability.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory method `withAttributes`. This is typically done during the bootstrap phase of an NPC asset builder, where the required validation rules for a given asset type are defined and collected.
- **Scope:** Transient and narrowly scoped. An instance exists only for the duration of a single asset's validation cycle. It holds no long-term state and is discarded once the validation is complete.
- **Destruction:** The object is lightweight and managed entirely by the Java garbage collector. No manual cleanup is required.

## Internal State & Concurrency
- **State:** Immutable. The internal fields, `parameter` and `attribute`, are final and set only during construction. The object's state cannot be modified after creation.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. A single instance can be safely shared and executed by multiple threads simultaneously without the need for external synchronization or locks. This is critical in a server environment where assets may be loaded and validated in parallel.

## API Surface
The public contract is minimal, consisting only of static factory and utility methods. The core validation logic is likely invoked via a method inherited from the parent Validator class, which is not part of the public API of this specific type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(parameter, attribute) | ExistsIfParameterSetValidator | O(1) | Static factory method. Constructs a new validator instance with the specified rule. |
| errorMessage(parameter, attribute) | String | O(1) | Static utility. Generates a standardized, human-readable error message for this validation rule. |

## Integration Patterns

### Standard Usage
This validator should be instantiated via its factory method and added to a collection of rules that are processed by a central validation coordinator or asset builder.

```java
// Example within an asset builder's configuration
// Assume 'builder' has a method to add validation rules.

builder.addValidator(
    ExistsIfParameterSetValidator.withAttributes("hasSpecialAbility", "specialAbilityPower")
);

builder.addValidator(
    ExistsIfParameterSetValidator.withAttributes("isRangedAttacker", "projectileType")
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private, preventing direct instantiation with `new`. This is an intentional design choice to enforce the use of the descriptive `withAttributes` factory method. Do not attempt to bypass this using reflection.
- **Stateful Logic:** Do not attempt to extend this class to add mutable state. Validators are expected to be stateless, reusable, and thread-safe rule definitions.

## Data Pipeline
This component acts as a gate within the broader NPC asset loading pipeline. It does not transform data but rather asserts its structural correctness.

> Flow:
> NPC Asset File (JSON/HOCON) -> Asset Parser -> Asset Builder -> **ExistsIfParameterSetValidator** -> Validation Engine -> Validated Asset or Error

