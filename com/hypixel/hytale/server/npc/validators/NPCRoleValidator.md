---
description: Architectural reference for NPCRoleValidator
---

# NPCRoleValidator

**Package:** com.hypixel.hytale.server.npc.validators
**Type:** Singleton

## Definition
```java
// Signature
public class NPCRoleValidator implements LateValidator<String> {
```

## Architecture & Concepts

The NPCRoleValidator is a specialized component within the server's data serialization and validation framework, known as the **Codec** system. Its primary function is to act as a bridge between the generic data parsing layer and the domain-specific logic of the NPC (Non-Player Character) system.

This class implements the `LateValidator` interface, which is a critical architectural choice. It signifies that this validation step occurs *after* the initial, context-free parsing of a configuration file. This "late" stage is essential because validating an NPC role requires querying the **NPCPlugin**, which must be fully initialized and have its role registry populated. A standard, early-stage validator would not have access to this runtime data.

The validator's secondary responsibility, exposed via the `updateSchema` method, is to enrich the schema metadata. By annotating a string field with a `HytaleAssetRef` of `NPCRole`, it provides crucial type information to the Hytale toolset. This allows, for example, an in-game editor or a content linter to understand that a given text field expects a valid NPC Role identifier, enabling features like autocompletion or static analysis.

In essence, this class ensures that any string intended to represent an NPC role in a configuration file corresponds to an actual, spawnable role known to the running server.

### Lifecycle & Ownership
- **Creation:** The singleton `INSTANCE` is instantiated by the JVM during class loading. This is a static, eager initialization.
- **Scope:** Application-scoped. The single instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is destroyed when the JVM shuts down. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The NPCRoleValidator is **stateless**. It does not maintain any internal data or cache. Its validation logic is entirely dependent on the state of the external `NPCPlugin`.
- **Thread Safety:** This class is inherently **thread-safe**. As it contains no mutable state, its methods can be invoked concurrently from multiple threads without risk of race conditions or data corruption. The responsibility for thread safety is delegated to the underlying `NPCPlugin`, which is expected to provide safe concurrent access to its role registry.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| acceptLate(String, ValidationResults, ExtraInfo) | void | O(1) | Performs the core validation logic. Delegates to `NPCPlugin` to verify the role string. If the role is invalid, an `IllegalArgumentException` is caught and its message is logged as a failure in the `ValidationResults`. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies the schema definition for a field, marking it as a reference to an `NPCRole` asset. This is for tooling and editor integration. |
| accept(String, ValidationResults) | void | O(1) | No-op. This method satisfies the base `Validator` interface but is not used in the late validation lifecycle. All logic is in `acceptLate`. |

## Integration Patterns

### Standard Usage
This validator is not designed to be invoked directly. It is configured declaratively within a `Schema` definition and is executed automatically by the Codec framework during data deserialization.

A developer would typically associate this validator with a string field in a component's schema definition.

```java
// Hypothetical schema definition
// This code is NOT part of the validator, but shows how it is used.

public class MyNPCComponent extends Component {
    public static final Schema SCHEMA = Schema.builder()
        .field("role", Schema.ofString().withValidator(NPCRoleValidator.INSTANCE))
        .build();

    private String role;
}
```
When an asset containing `MyNPCComponent` is loaded, the Codec system will automatically use the `NPCRoleValidator.INSTANCE` to validate the contents of the `role` field.

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Do not call `NPCRoleValidator.INSTANCE.acceptLate(...)` in your own code. This bypasses the schema-driven validation pipeline and couples your logic directly to the validator. Always rely on the Codec framework to execute validators.
- **Misunderstanding Lifecycle:** Attempting to validate data before the `NPCPlugin` has been fully initialized will result in validation failures or runtime exceptions. The use of `LateValidator` is intentional to prevent this, and the pattern should be respected.

## Data Pipeline
The validator operates as a single step within a larger data loading and validation pipeline.

> Flow:
> NPC Asset File (e.g., JSON) -> Server Asset Loading -> Codec Deserializer -> **NPCRoleValidator** -> `NPCPlugin.validateSpawnableRole` -> ValidationResults -> Component Instantiation

