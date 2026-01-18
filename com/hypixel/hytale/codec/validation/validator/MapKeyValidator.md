---
description: Architectural reference for MapKeyValidator
---

# MapKeyValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient Component

## Definition
```java
// Signature
public class MapKeyValidator<K> implements Validator<Map<K, ?>> {
```

## Architecture & Concepts
The MapKeyValidator is a specialized component within the Hytale Codec validation framework. Its sole responsibility is to enforce validation rules on the *keys* of a Map data structure, ignoring the values.

This class implements a **Composite Pattern**. It does not contain validation logic itself; instead, it holds a reference to another Validator instance and delegates the validation of each key to this composed validator. This design is fundamental to building complex and reusable validation chains. For instance, a MapKeyValidator can be configured with a StringValidator to ensure all keys in a configuration map adhere to a specific regex pattern.

Architecturally, it serves as a structural adapter, applying a validator designed for a single element (type K) to the key set of a collection (type Map). It also plays a crucial role in schema generation by translating its key constraints into rules for property names within an ObjectSchema.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its public constructor: `new MapKeyValidator(validator)`. It is typically instantiated by a higher-level factory or builder that assembles a complete validation graph for a specific data structure. It is not managed by a dependency injection container or service registry.
- **Scope:** The lifecycle is ephemeral and is strictly bound to the parent object that creates it. It persists only for the duration of a single, complete validation or schema-building operation.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the validation process that created it completes and all references to it are dropped. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state consists of a single reference to the composed `key` Validator. This state is **immutable** after construction. The validator instance provided in the constructor cannot be changed for the lifetime of the MapKeyValidator instance.
- **Thread Safety:** This class is conditionally thread-safe. Its safety is entirely dependent on the thread safety of the composed Validator instance it holds. As the MapKeyValidator itself performs no internal state mutation during its `accept` or `updateSchema` operations, it is safe for concurrent use *if and only if* the composed validator is also thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(Map, ValidationResults) | void | O(N) | Iterates through all N keys of the input map, applying the composed validator to each one. Populates the ValidationResults object with any failures. |
| updateSchema(SchemaContext, Schema) | void | O(1) | Modifies the target Schema to reflect the key constraints. **Warning:** Throws IllegalArgumentException if the target is not an ObjectSchema. |
| getKeyValidator() | Validator<K> | O(1) | Returns the composed validator used for key validation. |

## Integration Patterns

### Standard Usage
The MapKeyValidator is used to build more complex validators for map-like structures. It is never used in isolation.

```java
// Example: Validate a map to ensure all its keys are lowercase strings.
Validator<String> keyRule = new StringValidator().pattern("^[a-z]+$");
Validator<Map<String, ?>> mapValidator = new MapKeyValidator<>(keyRule);

Map<String, Integer> data = Map.of("validkey", 1, "InvalidKey", 2);
ValidationResults results = new ValidationResults();

mapValidator.accept(data, results);
// The 'results' object will now contain a validation failure for "InvalidKey".
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Schema Target:** Calling `updateSchema` with a Schema type that is not an ObjectSchema will cause a runtime crash. The validator's contract strictly requires an ObjectSchema to apply its property name constraints.
- **Null Composition:** Constructing a MapKeyValidator with a null validator will result in a NullPointerException during the `accept` or `updateSchema` calls. The composed validator is non-nullable.

## Data Pipeline
The MapKeyValidator acts as a dispatcher in two distinct data flows: runtime validation and schema generation.

**Validation Flow:**
> Flow:
> Input `Map<K, ?>` -> **MapKeyValidator** iterates `keySet()` -> For each key `k`, invokes `keyValidator.accept(k)` -> Failures are written to `ValidationResults`

**Schema Generation Flow:**
> Flow:
> Input `ObjectSchema` -> **MapKeyValidator** accesses `propertyNames` schema -> Invokes `keyValidator.updateSchema()` on the `propertyNames` schema -> The original `ObjectSchema` is mutated with new constraints

