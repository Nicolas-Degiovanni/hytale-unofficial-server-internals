---
description: Architectural reference for ArraysOneSetValidator
---

# ArraysOneSetValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class ArraysOneSetValidator extends Validator {
```

## Architecture & Concepts
The ArraysOneSetValidator is a specialized component within the server-side NPC asset validation framework. Its primary architectural role is to enforce a specific constraint on asset definitions: that out of a given pair of array-based attributes, at least one must be defined and contain at least one non-empty string element.

This class embodies the Strategy pattern, where the validation logic is encapsulated into a distinct, stateless object. It is designed to be a leaf component in a larger validation system, likely invoked by an asset parser or builder during the deserialization and verification of NPC configuration files. The design cleanly separates the pure validation logic (the static *validate* method) from the contextual metadata (the attribute names stored in an instance), allowing the parent validation framework to be data-driven and highly configurable.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand via the static factory methods *withAttributes*. This class is not a managed service and is not registered with any dependency injection context. It is typically instantiated by higher-level builder or configuration classes.
- **Scope:** Transient. The lifecycle of an ArraysOneSetValidator instance is extremely short. It is expected to exist only for the duration of a single asset's validation phase.
- **Destruction:** The object is immediately eligible for garbage collection once the validation check is complete. There are no persistent references or cleanup procedures required.

## Internal State & Concurrency
- **State:** The internal state consists of a single final field, *attributes*, which is an array of strings. This state is immutable after the object is constructed. The core validation logic is implemented in static methods and is therefore stateless.
- **Thread Safety:** This class is inherently thread-safe. Its immutable state and the pure-function nature of its static methods guarantee that it can be safely used in multi-threaded asset loading pipelines without any external synchronization.

## API Surface
The public contract is composed entirely of static factory and utility methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(String[] v1, String[] v2) | boolean | O(N+M) | Executes the core validation logic. Returns true if at least one of the input arrays contains a non-empty string. |
| formatErrorMessage(String, String, String) | String | O(1) | A static utility to generate a standardized, human-readable error message for validation failures. |
| withAttributes(String, String) | ArraysOneSetValidator | O(1) | Factory method to create a configured validator instance for two specific attribute names. |
| withAttributes(String[]) | ArraysOneSetValidator | O(1) | Factory method to create a configured validator instance for a collection of attribute names. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by a higher-level asset processing system. The static *validate* method is called directly with the data to be checked.

```java
// Within an asset loader or builder
String[] soundEvents = asset.getSoundEvents();
String[] particleEvents = asset.getParticleEvents();

boolean isValid = ArraysOneSetValidator.validate(soundEvents, particleEvents);

if (!isValid) {
    String error = ArraysOneSetValidator.formatErrorMessage("soundEvents", "particleEvents", asset.getName());
    throw new AssetValidationException(error);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to instantiate it via reflection would violate the design contract, which relies on the provided static factory methods.
- **Misinterpretation of Logic:** Do not assume this validator checks for null arrays. An empty array is considered valid if the corresponding peer array contains a non-empty string. The check is specifically for the *contents* of the arrays, not their existence.
- **Ignoring Formatter:** Avoid creating custom error messages. The static *formatErrorMessage* method provides consistent error reporting that is crucial for debugging asset configuration issues.

## Data Pipeline
The ArraysOneSetValidator acts as a synchronous processing and validation gate within a larger data flow. It does not source, sink, or transform data; it merely asserts its state.

> Flow:
> NPC Asset File (JSON/HOCON) -> Deserializer -> Raw Asset Object -> **ArraysOneSetValidator.validate()** -> Validation Result (Boolean) -> Asset Factory / Game Registry

