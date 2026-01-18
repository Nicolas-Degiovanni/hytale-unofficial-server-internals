---
description: Architectural reference for the Metadata interface, a contract for schema modification.
---

# Metadata

**Package:** com.hypixel.hytale.codec.schema.metadata
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface Metadata {
    void modify(Schema var1);
}
```

## Architecture & Concepts
The Metadata interface defines a contract for objects that apply modifications to a Schema configuration. It embodies a Strategy or Command pattern, where the specific logic for altering a schema is encapsulated within a concrete implementation of this interface.

This design decouples the core Schema definition from the various transformations, annotations, or augmentations it might undergo. Systems that process schemas, such as codec generators or data validators, can accept a collection of Metadata implementations to dynamically configure a base Schema before it is finalized and used. This allows for a highly modular and extensible data modeling pipeline where behaviors can be injected without altering the core schema-processing machinery.

A Metadata object is essentially a single, atomic operation to be performed on a Schema instance.

### Lifecycle & Ownership
As an interface, Metadata itself has no lifecycle. The following rules apply to its concrete implementations.

- **Creation:** Implementations are typically created as stateless, transient objects at the point where a schema modification pipeline is being assembled. They are often instantiated alongside the system that will consume them.
- **Scope:** The scope of a Metadata implementation is typically short-lived. It exists only for the duration of the schema modification process and is eligible for garbage collection immediately after its `modify` method has been invoked.
- **Destruction:** Managed by the Java Garbage Collector. There are no explicit cleanup or teardown requirements for implementations of this interface.

## Internal State & Concurrency
The interface itself is stateless. The following are strict guidelines for all implementations.

- **State:** Implementations of Metadata **should be stateless**. The `modify` method should operate exclusively on the provided Schema argument. Storing state within a Metadata object can lead to unpredictable behavior, especially if the same instance is reused across different schema modifications.

- **Thread Safety:** Implementations are **not required to be thread-safe**. The schema processing pipeline is expected to be a single-threaded, sequential operation. The system guarantees that a given Schema object will not be accessed by multiple threads while a Metadata object is modifying it.

## API Surface
The public contract consists of a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(N) | Applies an encapsulated modification to the provided Schema object. Complexity is dependent on the nature of the modification. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement the interface and pass an instance to a schema processing system. The system will then invoke the `modify` method at the appropriate stage in its pipeline.

```java
// 1. Define a concrete implementation of the Metadata contract.
public class AddVersionInfo implements Metadata {
    private final int version;

    public AddVersionInfo(int version) {
        this.version = version;
    }

    @Override
    public void modify(Schema schema) {
        // This is a hypothetical method on the Schema class
        schema.setVersion(this.version);
    }
}

// 2. Use it to configure a schema builder or processor.
Schema baseSchema = loadBaseSchema();
SchemaProcessor processor = new SchemaProcessor(baseSchema);

// 3. Apply the modification.
processor.applyMetadata(new AddVersionInfo(2));

Schema finalSchema = processor.build();
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not design implementations that rely on internal state changing between calls. Each `modify` call should be idempotent and self-contained.
- **Cross-Schema Contamination:** The `modify` method must *only* operate on the Schema instance passed to it. Reading from or writing to external or static state is a critical anti-pattern that breaks the encapsulation principle.
- **Order Dependency:** Avoid creating Metadata implementations that are implicitly dependent on the execution order of other Metadata objects. If ordering is required, the system consuming the Metadata should provide an explicit mechanism for it.

## Data Pipeline
The Metadata interface acts as a transformation step within a larger schema definition and processing pipeline.

> Flow:
> Base Schema Definition (from file or code) -> Schema Parser -> Schema Object -> **Metadata.modify()** -> Finalized Schema Object -> Codec Generator / Validator

