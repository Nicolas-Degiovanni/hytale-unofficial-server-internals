---
description: Architectural reference for ArraySchema
---

# ArraySchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Transient

## Definition
```java
// Signature
public class ArraySchema extends Schema {
```

## Architecture & Concepts
The ArraySchema class is a concrete implementation of the Schema contract, designed to define and enforce validation rules for array-like data structures. It is a critical component of Hytale's data-driven configuration system, which relies on a powerful Codec and Schema framework for serialization, deserialization, and validation.

This class functions as a declarative data model. An instance of ArraySchema does not perform validation itself; rather, it holds the metadata—such as item type, minimum/maximum length, and uniqueness constraints—that a higher-level validation engine uses to process data.

The static final field **CODEC** is the cornerstone of this class's integration into the engine. It is a BuilderCodec that defines how an ArraySchema object is itself encoded and decoded, typically from a BSON data source. This self-describing nature allows developers and designers to define complex data validation rules entirely within external configuration files, which are then loaded by the engine at runtime to construct an in-memory schema tree.

A key architectural feature is the polymorphic handling of the *items* property, managed by the internal and deprecated ItemOrItems codec. This allows a schema to specify either a single Schema for all array elements or an array of Schemas, where each element in the data array must conform to at least one of the provided schemas.

### Lifecycle & Ownership
-   **Creation:** ArraySchema instances are primarily created by the framework's deserialization pipeline. When a configuration file containing an array schema definition is loaded, the static CODEC field is invoked to instantiate and populate a new ArraySchema object. They can also be instantiated programmatically by developers when constructing schema definitions in code.
-   **Scope:** The lifetime of an ArraySchema object is bound to the parent Schema or SchemaContext that contains it. It is a transient object, existing as a node within a larger, in-memory schema tree. It is not a globally managed service.
-   **Destruction:** The object is reclaimed by the Java garbage collector when the schema tree it belongs to is no longer referenced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** Mutable. ArraySchema is a Plain Old Java Object (POJO) with public setters for its validation properties. Its state is intended to be configured upon creation and then treated as immutable.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization. It is designed to be created and configured in a single-threaded context, such as during the engine's initial asset loading phase.

    **WARNING:** Modifying an ArraySchema object after it has been registered with the validation system or while it is being used by a Codec on another thread will result in unpredictable behavior and potential race conditions. Treat instances as effectively immutable after their initial configuration.

## API Surface
The public API consists primarily of setters to configure the validation rules.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setItem(Schema items) | void | O(1) | Sets a single schema that all elements in the data array must conform to. |
| setItems(Schema... items) | void | O(1) | Sets an array of schemas. Elements in the data array must conform to one of these schemas. |
| setMinItems(Integer minItems) | void | O(1) | Sets the inclusive minimum number of items required in the data array. |
| setMaxItems(Integer maxItems) | void | O(1) | Sets the inclusive maximum number of items allowed in the data array. |
| setUniqueItems(boolean uniqueItems) | void | O(1) | If true, all items in the data array must be unique. |

## Integration Patterns

### Standard Usage
The primary use case is programmatic construction of a schema tree, which is then used to build a validator or a more complex codec.

```java
// Example: Define a schema for a game inventory hotbar, which must
// contain between 1 and 9 unique item identifiers (represented by a StringSchema).

// Assume StringSchema is another available Schema type.
StringSchema itemIdentifierSchema = new StringSchema();
itemIdentifierSchema.setPattern("hytale:item_[a-z_]+");

ArraySchema hotbarSchema = new ArraySchema();
hotbarSchema.setItem(itemIdentifierSchema);
hotbarSchema.setMinItems(1);
hotbarSchema.setMaxItems(9);
hotbarSchema.setUniqueItems(true);

// The 'hotbarSchema' object would then be added as a property
// to a larger schema, such as a PlayerStateSchema.
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never modify an ArraySchema instance after it has been passed to a Codec or validation engine. All configuration should occur before the schema is put into active use.
-   **Ignoring Deprecation:** The internal mechanism for handling single vs. multiple item schemas (ItemOrItems) is marked as deprecated. While it currently functions, building complex logic that relies on its specific implementation details is discouraged, as this internal pattern may be refactored or removed in future engine updates.

## Data Pipeline
ArraySchema is a blueprint used during the data validation stage. It does not process the data itself but provides the rules for another component to do so.

> **Schema Definition Flow:**
> BSON Configuration File -> **ArraySchema.CODEC** Deserialization -> In-Memory **ArraySchema** Object
>
> **Data Validation Flow:**
> Raw BSON Data -> Validation Engine (using **ArraySchema** rules) -> Validated Data or Validation Error

