---
description: Architectural reference for BuilderField
---

# BuilderField

**Package:** com.hypixel.hytale.codec.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderField<Type, FieldType> {
```

## Architecture & Concepts
The BuilderField class is the atomic unit of configuration within the Hytale serialization framework, specifically for codecs created via BuilderCodec. It is not intended for direct use but serves as a powerful internal descriptor that defines the complete behavior of a single field within a larger, complex object.

A BuilderCodec instance is fundamentally a collection of BuilderField objects. Each BuilderField encapsulates the entire lifecycle for one property, including:
-   **Serialization Key:** The name of the field in the serialized format (e.g., BSON or JSON), managed by the contained KeyedCodec.
-   **Data Binding:** High-performance function pointers (getter and setter) that map the serialized data to and from a Java object's field.
-   **Inheritance Logic:** Rules for how a field's value should be inherited from a parent object definition.
-   **Validation:** A chain of Validator instances that enforce constraints on the field's value during deserialization.
-   **Versioning:** The data format versions in which this field is active.
-   **Schema Generation:** Metadata and documentation strings used to auto-generate data schemas.

By encapsulating these concerns, BuilderField allows the BuilderCodec to be a simple orchestrator, iterating over its fields and delegating all field-specific operations. This design promotes a declarative, type-safe, and highly configurable approach to data serialization.

## Lifecycle & Ownership
-   **Creation:** A BuilderField is never instantiated directly. It is created exclusively through the fluent `BuilderField.FieldBuilder`, which is itself exposed by a `BuilderCodec.BuilderBase` during codec definition. The lifecycle begins when `FieldBuilder.add()` is called, which constructs the BuilderField and registers it with the parent `BuilderCodec.BuilderBase`.

-   **Scope:** The lifetime of a BuilderField is inextricably linked to its owning BuilderCodec. It is created once during codec definition and persists, immutable, for the entire application lifecycle or until the codec itself is garbage collected.

-   **Destruction:** The object is managed by the Java garbage collector. It is destroyed only when the BuilderCodec that contains it is no longer referenced. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** The BuilderField is **effectively immutable**. All of its core configuration fields are declared final and are set once in the constructor. This immutability is critical to the design, as it guarantees that a codec's behavior cannot be altered at runtime.

-   **Thread Safety:** The class is **inherently thread-safe**. Due to its immutable nature, a single BuilderField instance can be safely used by multiple threads simultaneously to perform encoding, decoding, and validation operations without requiring any external locks.

    **WARNING:** While the BuilderField itself is thread-safe, the objects it operates upon (the target object `Type` and the `ExtraInfo` context) may not be. Concurrency guarantees are the responsibility of the calling thread.

## API Surface
The primary API is invoked by the parent BuilderCodec during its orchestration of serialization and deserialization processes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(doc, t, info) | void | O(N) | Reads a value from the BsonDocument using the internal KeyedCodec and applies it to the target object `t` via `setValue`. Complexity is O(N) where N is the number of validators. |
| encode(doc, t, info) | void | O(1) | Retrieves a value from the target object `t` via the getter and writes it to the BsonDocument. |
| inherit(t, parent, info) | void | O(1) | Executes the specific inheritance logic for this field, applying values from a `parent` object to the target object `t`. |
| validate(t, info) | void | O(N) | Retrieves the field's value from object `t` and runs all associated validators against it. Complexity is O(N) where N is the number of validators. |
| setValue(t, value, info) | void | O(N) | The core mutation entry point. Runs all validators on the provided `value` before calling the setter to apply it to the target object `t`. |
| updateSchema(context, target) | void | O(M) | Populates a Schema object with metadata, validation rules, and documentation for this field. Complexity is O(M) where M is the number of validators and metadata entries. |

## Integration Patterns

### Standard Usage
A developer never interacts with a BuilderField instance directly. Its creation and configuration are handled entirely through the fluent builder pattern provided by BuilderCodec. The pattern involves defining a field, chaining configuration methods, and finally adding it to the codec builder.

```java
// Standard usage is declarative, within a BuilderCodec definition.
// The developer interacts with FieldBuilder, not BuilderField.
private static final Codec<MyComponent> CODEC = BuilderCodec.builder(MyComponent::new)
    .addField(
        "health",
        Codec.INT,
        MyComponent::setHealth,
        MyComponent::getHealth
    ) // This returns a FieldBuilder
    .setVersionRange(1, 5)
    .addValidator(Validators.range(0, 100))
    .documentation("The health of the component.")
    .add() // This creates the BuilderField and adds it to the codec
    .build();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is `protected` and should never be invoked directly. Attempting to bypass the `FieldBuilder` will result in an incomplete and non-functional field definition.
-   **Stateful Lambdas:** Providing stateful lambdas or method references for the getter, setter, or inherit functions is a severe anti-pattern. These functions may be executed concurrently and must not rely on or modify external state, as this breaks the thread-safety guarantees of the codec system.
-   **Ignoring `add()`:** Forgetting to call `add()` at the end of a field definition chain is a common error. This fails to register the configured BuilderField with the parent codec, and the field will be silently ignored during serialization and deserialization.

## Data Pipeline
BuilderField acts as a single, focused step within the larger data processing pipeline managed by BuilderCodec.

**Decode Pipeline Example:**
> Flow:
> BsonDocument -> `BuilderCodec.decode()` -> (Iteration) -> **`BuilderField.decode()`** -> `KeyedCodec.getOrNull()` -> **`BuilderField.setValue()`** -> (Validation Loop) -> `setter.accept()` -> Target Java Object Field

**Schema Generation Pipeline Example:**
> Flow:
> Schema Generation Request -> `BuilderCodec.updateSchema()` -> (Iteration) -> **`BuilderField.updateSchema()`** -> (Validator/Metadata Loop) -> `validator.updateSchema()` -> Mutated Schema Object

