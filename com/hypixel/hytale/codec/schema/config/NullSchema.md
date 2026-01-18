---
description: Architectural reference for NullSchema
---

# NullSchema

**Package:** com.hypixel.hytale.codec.schema.config
**Type:** Singleton

## Definition
```java
// Signature
public class NullSchema extends Schema {
```

## Architecture & Concepts
The NullSchema class is an implementation of the **Null Object** design pattern within the Hytale codec and configuration system. Its sole purpose is to provide a concrete, non-null representation for the absence of a schema definition.

This class acts as a sentinel value. Instead of using a Java null reference, which can lead to NullPointerExceptions and requires explicit null checks, the system uses the static NullSchema.INSTANCE. This allows for more robust and streamlined schema processing logic, as consumers of a Schema object can treat a NullSchema instance like any other schema without special case handling for nulls.

It is a foundational, structural component used by higher-level configuration builders and parsers to signify that a particular data field has no defined structure or is intentionally empty.

## Lifecycle & Ownership
- **Creation:** The single NullSchema.INSTANCE is instantiated eagerly by the Java Virtual Machine during class loading. This is a compile-time, static initialization.
- **Scope:** As a static final singleton, the instance persists for the entire lifetime of the application. Its scope is global.
- **Destruction:** The object is eligible for garbage collection only when the application's ClassLoader is unloaded, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** NullSchema is **deeply immutable**. It contains no instance fields and its parent class, Schema, provides no mutable state. Its identity is its state.
- **Thread Safety:** This class is inherently thread-safe. As an immutable singleton, it can be safely accessed and shared across any number of threads without requiring locks or any other synchronization mechanisms.

## API Surface
The public contract of NullSchema is defined by its static constants, not by instance methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | NullSchema | O(1) | The canonical, globally shared instance of NullSchema. |
| CODEC | BuilderCodec | O(1) | The codec responsible for serializing and deserializing the NullSchema type. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to use the static INSTANCE field as a placeholder or terminal value when constructing or interpreting schema definitions. It signals the end of a recursive definition or the absence of a complex type.

```java
// Example: Using NullSchema as a default in a configuration map
Map<String, Schema> schemas = new HashMap<>();
Schema userSchema = schemas.getOrDefault("user_profile", NullSchema.INSTANCE);

// The consumer can now operate on userSchema without a null check
processSchema(userSchema);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a new instance via `new NullSchema()`. This defeats the purpose of the singleton pattern, creates unnecessary object garbage, and can lead to incorrect behavior in systems that rely on reference equality checks (e.g., `if (schema == NullSchema.INSTANCE)`).
- **Subclassing:** Do not extend NullSchema. It is a final, concrete implementation of a pattern and provides no extension points or overridable behavior.

## Data Pipeline
NullSchema does not actively process data. Instead, it is a descriptor that *directs* the data pipeline. When the codec system encounters a NullSchema, it interprets it as a signal that no data is present or expected for the corresponding field.

> Flow:
> Configuration Parser -> Schema Builder -> **NullSchema.INSTANCE** -> Codec Engine -> Skips Field in Serialized Output

