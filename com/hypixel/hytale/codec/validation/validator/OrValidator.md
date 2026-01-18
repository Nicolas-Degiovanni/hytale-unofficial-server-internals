---
description: Architectural reference for OrValidator
---

# OrValidator

**Package:** com.hypixel.hytale.codec.validation.validator
**Type:** Transient

## Definition
```java
// Signature
public class OrValidator<T> implements Validator<T> {
```

## Architecture & Concepts
The OrValidator is a composite validator that implements a logical OR operation. It acts as a container for an array of other Validator instances and succeeds if **at least one** of its child validators succeeds. This component is fundamental to the Hytale codec system for defining flexible data structures where a field can conform to one of several possible schemas.

Architecturally, OrValidator embodies the Composite design pattern. It allows client code to treat a group of validators as a single validation unit. Its primary role is to enable polymorphism in data definitions, such as a configuration value that can be either a single string or a list of strings.

The class has a dual responsibility:
1.  **Data Validation:** During deserialization or data processing, the `accept` method evaluates an input object against its child validators in sequence, short-circuiting and succeeding on the first valid path.
2.  **Schema Generation:** The `updateSchema` method contributes to building a formal schema definition (e.g., a JSON Schema). It populates the `anyOf` field in the target schema, which directly corresponds to the logical OR behavior of the validator.

## Lifecycle & Ownership
-   **Creation:** OrValidator instances are not intended for direct instantiation by end-users. They are constructed by higher-level components within the codec framework, such as a `BuilderCodec`, when a schema definition requires a choice between multiple structures.
-   **Scope:** The lifetime of an OrValidator is tied to the parent `BuilderCodec` or validation chain that created it. It is a transient, stateless object that exists only to encapsulate a specific validation rule.
-   **Destruction:** The object is managed by the Java garbage collector and is reclaimed when the codec or schema it belongs to is no longer in scope. No explicit cleanup is required.

## Internal State & Concurrency
-   **State:** The OrValidator is effectively immutable. Its only internal state is the `validators` array, which is marked as `final` and is provided exclusively at construction time. It holds no per-validation state; all necessary context is passed as arguments to its methods.
-   **Thread Safety:** This class is inherently thread-safe. As its internal state is immutable, a single OrValidator instance can be safely shared and executed by multiple threads simultaneously.

    **WARNING:** Thread safety guarantees apply only to the OrValidator instance itself. The `ValidationResults` object passed into the `accept` method is stateful and must not be shared across threads without external synchronization. The implementation temporarily manipulates the state of the `ValidationResults` object, a pattern which is safe only when the object is confined to a single thread of execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T t, ValidationResults results) | void | O(N) | Sequentially tests the input `t` against each child validator. Returns on the first success. If all fail, it aggregates all validation errors. |
| updateSchema(SchemaContext context, Schema target) | void | O(N) | Modifies the target Schema by populating its `anyOf` property. Each entry in `anyOf` is a schema generated from a corresponding child validator. |

## Integration Patterns

### Standard Usage
The OrValidator is an internal building block of the codec system and is not typically used directly. Developers interact with it implicitly through a builder API that constructs the validation graph.

```java
// Conceptual Example: A builder creates an OrValidator under the hood
// to allow a field to be either a String or an Integer.
Codec<JsonElement> stringOrIntCodec = Codecs.or(
    Codecs.STRING,
    Codecs.INTEGER
);

// When stringOrIntCodec.validate() is called, an OrValidator
// containing a StringValidator and an IntegerValidator is executed.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid `new OrValidator()`. The codec system's builders are responsible for constructing the validator graph correctly. Manual creation bypasses framework setup and can lead to an incomplete or incorrect validation chain.
-   **Empty Validator Array:** Constructing an OrValidator with an empty array of child validators creates a rule that can never succeed. This will cause validation to fail unconditionally for any input.
-   **Misinterpreting Failure Reports:** When validation fails, the `ValidationResults` object will contain a collection of errors from **all** attempted paths. Do not assume the first error is the only problem. The aggregated list must be interpreted as "the input failed to match schema A for these reasons, and it also failed to match schema B for these other reasons."

## Data Pipeline
The OrValidator functions as a branching point in both the data validation and schema generation pipelines.

**Validation Flow**
> Input Object (`T`) -> `OrValidator.accept()` -> Tries `Validator[0]`, `Validator[1]`, ... -> Short-circuits on first success OR aggregates all failures -> `ValidationResults`

**Schema Generation Flow**
> Parent `Schema` -> `OrValidator.updateSchema()` -> Generates sub-schemas from `Validator[0]`, `Validator[1]`, ... -> Populates `anyOf` field in Parent `Schema` -> Modified Parent `Schema`

