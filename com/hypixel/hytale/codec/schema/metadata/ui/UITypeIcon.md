---
description: Architectural reference for UITypeIcon
---

# UITypeIcon

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient Value Object

## Definition
```java
// Signature
public class UITypeIcon implements Metadata {
```

## Architecture & Concepts
The UITypeIcon class is a specific implementation of the Metadata interface, designed to encapsulate a single piece of information: the asset path for a UI icon. It functions as a *Command* or *Strategy* object within the engine's configuration schema system.

Its primary role is to decouple the source of configuration data (e.g., a JSON asset definition file) from the in-memory Schema object that represents the final, consolidated configuration. Instead of having a parser directly manipulate the Schema, the parser creates a list of Metadata objects, including UITypeIcon instances. A separate processing stage then iterates through this list, applying each piece of metadata to the Schema in a controlled manner. This pattern provides a clear separation of concerns between data parsing and data application.

## Lifecycle & Ownership
- **Creation:** Instances are not intended to be created manually by game logic developers. They are instantiated by a deserialization or configuration loading framework when it encounters a corresponding key, such as *uiTypeIcon*, in an asset definition file.

- **Scope:** The lifecycle of a UITypeIcon instance is extremely short. It is created during the asset loading phase, used once to apply its value to a Schema object, and is then immediately eligible for garbage collection. It is a transient, single-use object.

- **Destruction:** There is no explicit destruction. The object is reclaimed by the Java garbage collector once all references to it are dropped, which typically occurs after the schema modification loop completes.

## Internal State & Concurrency
- **State:** The UITypeIcon is **Immutable**. Its internal state, the *icon* string, is a private final field set only once via the constructor. The object's state cannot be changed after creation.

- **Thread Safety:** This class is inherently **Thread-Safe**. Due to its immutability, an instance can be safely read by multiple threads without any external synchronization. The *modify* method operates on a Schema object passed as a parameter; thread safety of the overall operation depends on the implementation of the Schema class, not the UITypeIcon itself.

## API Surface
The public contract consists solely of the constructor and the implementation of the Metadata interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UITypeIcon(String) | constructor | O(1) | Constructs a new metadata object with the specified icon path. |
| modify(Schema) | void | O(1) | Applies the stored icon path to the target Schema object. This is a mutating operation on the passed-in Schema. |

## Integration Patterns

### Standard Usage
UITypeIcon is not used in isolation. It is processed as part of a collection of Metadata objects that are applied sequentially to a Schema.

```java
// A central Schema object is prepared for an asset
Schema targetSchema = new Schema();

// A list of metadata is loaded from a configuration source.
// The loader would create new UITypeIcon("path/to/icon") internally.
List<Metadata> metadataList = configurationLoader.loadMetadataForAsset("my_asset");

// A processor applies all loaded metadata to the schema
for (Metadata metadata : metadataList) {
    metadata.modify(targetSchema);
}

// The targetSchema now contains the UI icon path.
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Avoid creating instances with *new UITypeIcon()*. This class is designed to be a product of automated configuration parsing, not manual construction in game logic.

- **State Storage:** Do not retain references to UITypeIcon instances after the schema loading process is complete. They are lightweight, transient objects and should not be used as a long-term data store.

- **Null Data:** Passing a null string to the constructor is not supported and will lead to undefined behavior or a NullPointerException when the *modify* method is eventually called.

## Data Pipeline
The UITypeIcon acts as a temporary data carrier in a larger configuration pipeline.

> Flow:
> Asset Definition File (e.g., JSON) -> Deserializer -> **new UITypeIcon("...")** -> Metadata Processor -> Schema Object State Update -> UI Rendering System

