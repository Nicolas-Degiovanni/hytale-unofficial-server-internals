---
description: Architectural reference for StringValidator
---

# StringValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Abstract Strategy

## Definition
```java
// Signature
public abstract class StringValidator extends Validator {
```

## Architecture & Concepts
The StringValidator class is an abstract base class that establishes a formal contract for all string-based validation logic within the server-side NPC asset pipeline. It embodies the Strategy Pattern, allowing for the definition of discrete, interchangeable validation rules that can be applied to string properties during asset compilation and loading.

This component is a foundational element of the engine's data integrity and schema enforcement system. Its primary role is to ensure that string values within NPC definition files (e.g., names, identifiers, script references) conform to predefined engine or game-specific constraints. By centralizing the validation contract, it promotes consistency and simplifies the creation of new validation rules.

## Lifecycle & Ownership
- **Creation:** Concrete implementations of StringValidator are instantiated by the asset validation framework, typically via a factory or reflection, when an asset build process is initiated. They are not intended for manual instantiation.
- **Scope:** The lifecycle of a StringValidator instance is ephemeral. It is created on-demand for a specific validation task and exists only for the duration of that operation.
- **Destruction:** Instances are eligible for garbage collection immediately after the validation check is complete and its result has been processed. They hold no long-term state and are not managed by any persistent context.

## Internal State & Concurrency
- **State:** The StringValidator base class is stateless. Subclasses are **strictly expected** to be stateless as well. Storing state within a validator instance is a severe anti-pattern that breaks reusability and introduces thread-safety issues.
- **Thread Safety:** This class is inherently thread-safe. All concrete implementations **must** be designed to be thread-safe and re-entrant. The asset building pipeline may process multiple assets in parallel, and a single validator instance may be used concurrently across multiple threads to validate different data sources.

## API Surface
The public contract consists entirely of abstract methods that must be implemented by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String var1) | boolean | Varies | Executes the core validation logic against the input string. Returns true if valid, false otherwise. |
| errorMessage(String var1) | String | O(1) | Generates a generic, context-free error message for a failed validation. |
| errorMessage(String var1, String var2) | String | O(1) | Generates a context-aware error message. Typically, var1 is the invalid value and var2 is the property name or location. |

## Integration Patterns

### Standard Usage
Developers should extend this class to implement a specific validation rule. The framework will then discover and apply this validator during the asset processing stage.

```java
// Example of a concrete implementation
public class NonEmptyStringValidator extends StringValidator {
    @Override
    public boolean test(String value) {
        return value != null && !value.trim().isEmpty();
    }

    @Override
    public String errorMessage(String value) {
        return "The provided string cannot be empty.";
    }

    @Override
    public String errorMessage(String value, String context) {
        return "The property '" + context + "' cannot be empty.";
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not add fields to subclasses that store per-validation state. This will break in a multi-threaded environment.
- **Complex Logic in Constructors:** Validators should be lightweight and cheap to construct. Defer all complex operations to the test method.
- **Ignoring Contextual Errors:** Failing to implement the two-argument errorMessage method meaningfully will result in poor debugging experiences for content creators.

## Data Pipeline
StringValidator sits within the server's asset ingestion pipeline, acting as a gatekeeper for data quality before assets are loaded into the game world.

> Flow:
> NPC Definition File (JSON/HOCON) -> Asset Deserializer -> Asset Validation Service -> **StringValidator** -> Validation Result (Pass/Fail) -> Asset Instantiation

