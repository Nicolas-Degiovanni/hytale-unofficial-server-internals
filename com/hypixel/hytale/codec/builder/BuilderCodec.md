---
description: Architectural reference for BuilderCodec
---

# BuilderCodec

**Package:** com.hypixel.hytale.codec.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderCodec<T> implements Codec<T>, DirectDecodeCodec<T>, RawJsonCodec<T>, InheritCodec<T>, ValidatableCodec<T> {
```

## Architecture & Concepts

The BuilderCodec is the cornerstone of Hytale's data serialization and deserialization framework. It provides a highly configurable, type-safe, and performant mechanism for converting Java objects to and from BSON and raw JSON representations. This class is not a generic serialization library; it is a purpose-built system designed for game asset configuration, network packets, and save data, with first-class support for versioning, inheritance, and schema generation.

Its core design philosophy is to define an object's serialization contract declaratively using a fluent builder pattern. This approach avoids the performance overhead and runtime fragility of reflection-based systems. Each serializable class typically defines a static final BuilderCodec instance that describes its own data layout.

Key architectural features include:

*   **Declarative Field Mapping:** Developers explicitly map data fields using the builder, providing setters, getters, and a corresponding Codec for the field's type. This creates a rigid, compile-time-verified contract for data shapes.
*   **Advanced Versioning:** BuilderCodec has a sophisticated versioning system. Fields can be defined to exist only within specific version ranges (`minVersion`, `maxVersion`). This allows the engine to gracefully handle legacy data formats during upgrades without manual data migration scripts. The entire object can also be versioned, which is encoded directly into the serialized output.
*   **Inheritance Model:** The `parentCodec` mechanism allows codecs to inherit fields and behaviors from a base codec. This is critical for asset polymorphism, where a specialized asset (e.g., an Iron Sword) can inherit properties from a base asset (e.g., a generic Sword) and only define its own unique fields.
*   **High-Performance JSON Parsing:** For JSON, the system uses a custom `RawJsonReader` and a specialized `StringTreeMap` data structure. This allows for direct, low-allocation parsing of JSON streams without creating intermediate `Map` or `JsonObject` representations, which is critical for performance-sensitive applications.
*   **Schema Generation:** The `toSchema` method can introspect a codec's configuration to generate a formal `ObjectSchema`. This schema is used by Hytale's tooling, such as the game editor, to provide auto-completion, validation, and rich UI for editing asset files.

The BuilderCodec acts as the translation layer between the raw, untyped data on disk or from the network and the strongly-typed Java object model used by the game engine.

### Lifecycle & Ownership
-   **Creation:** BuilderCodec instances are constructed exclusively through the static `builder()` or `abstractBuilder()` methods, culminating in a call to `build()`. They are intended to be defined once as `static final` constants within the class they are responsible for serializing.

-   **Scope:** An instance of BuilderCodec is a stateless configuration object. Once built, it is effectively immutable. It should persist for the entire lifetime of the application. It does not hold any per-session or per-request state.

-   **Destruction:** Instances are garbage collected by the JVM during application shutdown. There is no manual destruction or resource cleanup required.

## Internal State & Concurrency
-   **State:** A BuilderCodec instance is immutable after construction. All its configuration fields, such as `entries`, `parentCodec`, and `validator`, are final or set only within the constructor. It does not cache any data related to the objects it processes.

-   **Thread Safety:** The BuilderCodec is inherently thread-safe. Its methods (`decode`, `encode`, etc.) are pure functions with respect to the codec's own state. All mutable, context-sensitive information required during a serialization or deserialization operation (e.g., object version, validation results, key path) is passed via the `ExtraInfo` parameter. The engine typically uses a `ThreadLocal` instance of `ExtraInfo` to ensure that concurrent operations are fully isolated.

    **WARNING:** While the BuilderCodec itself is thread-safe, the object supplier and field accessors provided during its construction must also be thread-safe if the codec is used concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder(Class, Supplier) | Builder | O(1) | Creates a new fluent builder to define a codec for a concrete class. |
| abstractBuilder(Class) | Builder | O(1) | Creates a builder for an abstract class that cannot be instantiated directly but can serve as a parent. |
| decode(BsonValue, ExtraInfo) | T | O(N) | Deserializes a BSON value into a new object of type T. N is the number of fields. |
| encode(T, ExtraInfo) | BsonDocument | O(N) | Serializes an object of type T into a BSON document. N is the number of fields. |
| decodeJson(RawJsonReader, ExtraInfo) | T | O(N) | Deserializes a JSON stream into a new object of type T using the high-performance path. |
| inherit(T, T, ExtraInfo) | void | O(N) | Copies inheritable field values from a parent object to a child object. |
| validate(T, ExtraInfo) | void | O(N) | Runs validation logic defined for the codec and its fields against the given object. |
| toSchema(SchemaContext) | ObjectSchema | O(N) | Generates a JSON Schema definition for use in external tooling. |

## Integration Patterns

### Standard Usage

The canonical pattern is to define a `static final` codec within the target class. This makes the serialization contract a self-contained part of the data model.

```java
// In a hypothetical "ModelComponent" class
public class ModelComponent {
    public static final Codec<ModelComponent> CODEC = BuilderCodec.builder(ModelComponent.class, ModelComponent::new)
        .versioned()
        .append(
            new KeyedCodec<>("Model", ASSET_REFERENCE),
            ModelComponent::setModel,
            ModelComponent::getModel
        ).build();

    private AssetReference<Model> model;

    // getters and setters...
}

// Elsewhere, to use the codec:
ExtraInfo info = ExtraInfo.THREAD_LOCAL.get();
ModelComponent component = ModelComponent.CODEC.decode(bsonDocument, info);
```

### Anti-Patterns (Do NOT do this)
-   **Dynamic Instantiation:** Do not create BuilderCodec instances on the fly. They are heavyweight configuration objects. Defining them as static constants ensures they are created only once.
    ```java
    // BAD: Creates a new codec on every call
    public ModelComponent load(BsonDocument doc) {
        Codec<ModelComponent> codec = BuilderCodec.builder(...).build();
        return codec.decode(doc, ...);
    }
    ```
-   **Overlapping Version Ranges:** When defining versioned fields, ensure their version ranges do not overlap for the same key. This will throw an `IllegalArgumentException` at build time.
-   **Ignoring ExtraInfo:** The `ExtraInfo` parameter is critical for correct versioning, validation, and error reporting. Do not pass a default or empty instance if context is available.
-   **Modifying State in Getters:** Getters provided to the builder should be pure functions and not modify the state of the object. The serialization system may call them multiple times and expects consistent results.

## Data Pipeline

The BuilderCodec is a central component in the data transformation pipeline. It handles both inbound (deserialization) and outbound (serialization) data flows.

### Deserialization Flow
> Flow:
> Raw BSON / JSON Stream -> `decode` / `decodeJson` -> **BuilderCodec** (with `ExtraInfo`) -> New Java Object -> `afterDecode` hooks -> Validation -> Fully Hydrated Object

During this process, the BuilderCodec reads the version from the raw data, selects the appropriate field definitions for that version, and populates a new object instance created by the provided `Supplier`.

### Serialization Flow
> Flow:
> Java Object -> `encode` -> **BuilderCodec** (with `ExtraInfo`) -> BSON Document Construction -> Raw BSON

In this flow, the BuilderCodec iterates through its configured fields, calls the getter for each, uses the corresponding child codec to serialize the value, and assembles the final `BsonDocument`. If versioned, it injects the current `codecVersion` into the output.

