---
description: Architectural reference for CosmeticAssetValidator
---

# CosmeticAssetValidator

**Package:** com.hypixel.hytale.server.core.cosmetics
**Type:** Transient Utility

## Definition
```java
// Signature
public class CosmeticAssetValidator implements Validator<String> {
```

## Architecture & Concepts
The **CosmeticAssetValidator** is a specialized component within Hytale's data serialization and validation framework, known as the Codec system. Its primary function is to enforce referential integrity for cosmetic asset identifiers.

This class acts as a bridge between the abstract schema definition layer and the concrete, runtime state of the **CosmeticRegistry**. When the server deserializes data (such as player configuration or entity definitions), this validator ensures that any string field designated as a cosmetic asset points to a valid, registered cosmetic of a specific **CosmeticType**. For example, it can verify that a string "hytale:dev_helmet" is a valid and registered HAT cosmetic.

A secondary, but critical, function is schema enrichment via the **updateSchema** method. This annotates the data schema with Hytale-specific metadata, signaling to external tools—such as the Hytale Studio editor or network protocol generators—that a given string field represents a cosmetic asset. This allows for features like dropdown selectors in an editor UI or optimized network serialization.

## Lifecycle & Ownership
- **Creation:** Instances are created by the Codec framework during the schema construction phase. A developer does not instantiate this class directly in game logic. Instead, it is configured declaratively within a schema definition, which then triggers its instantiation by the framework.
- **Scope:** The lifetime of a **CosmeticAssetValidator** instance is typically tied to the lifecycle of the **Schema** object it is associated with. It is a short-lived, stateless object used during a validation or schema-building process.
- **Destruction:** The object is eligible for garbage collection as soon as the validation operation completes and the parent **Schema** object is no longer referenced.

## Internal State & Concurrency
- **State:** The **CosmeticAssetValidator** is effectively immutable. Its only internal state is the final **CosmeticType** field, which is set at construction and cannot be changed. The class does not cache any data; it performs a fresh lookup against the global **CosmeticRegistry** on every validation call.

- **Thread Safety:** This class is inherently thread-safe. As an immutable object, it can be safely shared and executed across multiple threads.

    **WARNING:** While the validator itself is thread-safe, its correctness depends on the thread safety of the global **CosmeticsModule** and **CosmeticRegistry** it accesses. It is assumed that the registry is designed for safe concurrent reads. Any mutation to the registry while a validation is in progress may lead to non-deterministic behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CosmeticAssetValidator(type) | constructor | O(1) | Creates a new validator for a specific **CosmeticType**. |
| accept(asset, results) | void | O(1) | Validates that the asset string exists in the **CosmeticRegistry**. Fails the validation if the key is not found. Complexity is dependent on the registry's map lookup. |
| updateSchema(context, target) | void | O(1) | Annotates the target **StringSchema** with metadata about the cosmetic type. |

## Integration Patterns

### Standard Usage
A developer will almost never interact with this class directly. Its usage is implicit and driven by the schema definition system. The following conceptual example illustrates how it might be configured.

```java
// Conceptual schema definition
// This code would be part of the framework, not typical game logic.

Schema playerProfileSchema = new ObjectSchema();

// Define a field for the player's hat
StringSchema hatField = new StringSchema();

// Attach the validator for the HAT cosmetic type
hatField.addValidator(new CosmeticAssetValidator(CosmeticType.HAT));

playerProfileSchema.addField("equippedHat", hatField);

// Later, when data is validated, the framework will invoke the validator.
ValidationResults results = new ValidationResults();
playerProfileSchema.validate(playerData, results); // The validator's accept method is called here.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Logic:** Do not create instances of **CosmeticAssetValidator** within game systems to perform one-off checks. This bypasses the Codec framework and couples your logic directly to the validator's implementation. Use the **CosmeticRegistry** directly for such lookups.
- **Reusing for Different Types:** An instance is bound to a single **CosmeticType**. Do not attempt to reuse it to validate assets of a different type.

## Data Pipeline
The validator sits within the data deserialization and validation pipeline. It is invoked by the Codec framework after raw data has been parsed but before it is accepted as valid game state.

> Flow:
> Raw Data (e.g., JSON) -> Codec Deserializer -> Schema Traversal -> **CosmeticAssetValidator.accept()** -> ValidationResults -> Validated Game Object

