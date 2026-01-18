---
description: Architectural reference for InstanceValidator
---

# InstanceValidator

**Package:** com.hypixel.hytale.builtin.instances
**Type:** Singleton

## Definition
```java
// Signature
public class InstanceValidator implements Validator<String> {
```

## Architecture & Concepts
The InstanceValidator is a specialized component within the Hytale Codec and Schema validation framework. Its sole responsibility is to verify that a given string value corresponds to a valid, registered "Instance" asset.

This class acts as a rule or a constraint that can be attached to a schema definition. When the codec system parses a configuration file (e.g., a JSON or HOCON asset) against a schema, it invokes this validator for any field designated as an Instance asset reference. This ensures data integrity at asset load time, preventing runtime errors caused by references to non-existent or misspelled instance assets.

The secondary method, updateSchema, provides metadata to the schema itself. This allows higher-level systems, such as the Hytale Model Editor or other content creation tools, to understand the *intent* of the field. For example, a UI could use this information to render a searchable dropdown list of all available Instance assets instead of a plain text input field, dramatically improving the content creation workflow.

## Lifecycle & Ownership
- **Creation:** The single instance of this class is created eagerly by the JVM during class loading via the `public static final INSTANCE` field. This is a classic eager-initialized singleton pattern.
- **Scope:** As a static singleton, its lifecycle is tied to the application's ClassLoader. It persists for the entire duration of the application session.
- **Destruction:** The object is eligible for garbage collection only when its ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** The InstanceValidator is **stateless**. It contains no mutable instance fields. Its validation logic is entirely dependent on the external state managed by the InstancesPlugin, which holds the registry of known instance assets.
- **Thread Safety:** This class is inherently **thread-safe**. Because it is stateless, multiple threads can safely invoke its methods on the shared `INSTANCE` without risk of race conditions or data corruption. The thread safety of the underlying `InstancesPlugin.doesInstanceAssetExist` call is a separate concern, but this validator introduces no concurrency issues of its own.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(String s, ValidationResults results) | void | O(1) | Validates that an instance asset with the given name exists. On failure, it mutates the provided ValidationResults object by calling its fail method. |
| updateSchema(SchemaContext context, Schema target) | void | O(1) | Annotates a schema definition, marking the target field as a custom Hytale asset reference of type "Instance". |

## Integration Patterns

### Standard Usage
The InstanceValidator is not intended to be invoked directly in game logic. Instead, it is registered declaratively within a schema definition. The codec framework then invokes it automatically during the data parsing and validation process.

```java
// Hypothetical usage during schema construction for a custom component
// This code would run as part of the engine's asset definition loading.

Schema customComponentSchema = Schema.builder()
    .field("worldEntryPoint", String.class, field -> {
        // Attach the validator to ensure any value for "worldEntryPoint"
        // is a valid and existing instance asset name.
        field.addValidator(InstanceValidator.INSTANCE);
    })
    .build();

// The codec system will now use this schema to validate assets.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InstanceValidator()`. The class is a singleton and provides a static `INSTANCE` field for this purpose. Creating new instances is wasteful and defeats the design pattern.
- **Misuse for General Asset Validation:** This validator is highly specific to "Instance" assets via its dependency on InstancesPlugin. Do not use it to validate other asset types like textures, sounds, or models. Doing so will always fail and is semantically incorrect.

## Data Pipeline
The InstanceValidator functions as a gate within the asset and configuration loading pipeline. It intercepts string values from raw data files and validates them against a central asset registry before they can be used by the runtime.

> Flow:
> Raw Config File (e.g., zone.json) -> Hytale Codec Parser -> **InstanceValidator.accept()** -> ValidationResults -> Asset Loading System or Editor UI Feedback

