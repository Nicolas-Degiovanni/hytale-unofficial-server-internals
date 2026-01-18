---
description: Architectural reference for UIEditorSectionStart
---

# UIEditorSectionStart

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient

## Definition
```java
// Signature
public class UIEditorSectionStart implements Metadata {
```

## Architecture & Concepts
The UIEditorSectionStart class is a specialized **Metadata Directive** used within the Hytale schema processing framework. It functions as a command object that encapsulates a single, specific instruction: to begin a new collapsible section within a dynamically generated user interface, such as an in-game editor or property inspector.

This class is a concrete implementation of the Metadata interface, adhering to a system where a collection of Metadata objects are applied sequentially to a central Schema object. This pattern allows for declarative and extensible UI and configuration definitions, where behavior is composed from a series of simple, single-purpose directives. UIEditorSectionStart's sole responsibility is to signal the start of a logical grouping of subsequent properties, identified by a human-readable title.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level parser or builder, typically when deserializing a configuration file (e.g., JSON, YAML). A new instance is created for each "section start" directive encountered in the source data.
- **Scope:** Extremely short-lived. An instance exists only for the brief moment it is needed to invoke its `modify` method on a Schema object during the configuration build process.
- **Destruction:** The object holds no external references and is immediately eligible for garbage collection after the `modify` method completes. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal state consists of a single `final` field, `title`, which is set exclusively at construction time. The object cannot be changed after it is created.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. An instance can be safely passed between threads. However, the `modify` operation it performs is **not atomic** and mutates the state of the provided Schema object. The caller is responsible for ensuring that the target Schema is accessed in a thread-safe manner.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(1) | Injects the section title into the target Schema's UI state. This is a mutating operation on the input parameter. |

## Integration Patterns

### Standard Usage
UIEditorSectionStart is not intended to be used in isolation. It is designed to be part of a collection of Metadata objects that are processed in order to configure a Schema.

```java
// A processor iterates through a list of metadata directives
Schema targetSchema = new Schema();
List<Metadata> directives = loadDirectivesFromConfig(); // Assume this loads various Metadata types

for (Metadata directive : directives) {
    // Each directive, including UIEditorSectionStart, applies its modification
    directive.modify(targetSchema);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Reuse:** Do not retain and reuse instances of UIEditorSectionStart. They are lightweight, immutable value objects that should be created as needed during configuration parsing.
- **Manual Invocation:** Avoid calling `modify` outside of a controlled schema build process. The order of Metadata application is critical, and manual calls can lead to an inconsistent or corrupt Schema state.

## Data Pipeline
This class acts as a single step in a larger schema configuration pipeline. Data flows from a static definition, is instantiated as this object, and then modifies a target schema.

> Flow:
> Configuration Source (e.g., JSON) -> Schema Deserializer -> **UIEditorSectionStart Instance** -> Schema Build Loop -> Modified Schema Object

