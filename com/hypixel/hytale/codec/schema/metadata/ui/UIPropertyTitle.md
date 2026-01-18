---
description: Architectural reference for UIPropertyTitle
---

# UIPropertyTitle

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient

## Definition
```java
// Signature
public class UIPropertyTitle implements Metadata {
```

## Architecture & Concepts
The UIPropertyTitle class is a specific implementation of the Metadata interface, designed to encapsulate a single piece of configuration: the display title for a UI property. It operates as a Command object within the schema processing framework. Its sole responsibility is to carry the title string and apply it to a target Schema object when invoked.

This class is a fundamental building block in a declarative configuration system. Instead of directly manipulating a Schema object, various metadata components like UIPropertyTitle are instantiated by a parser or builder. These components are then applied in sequence to a Schema, ensuring a consistent and decoupled modification process. This pattern allows the core Schema to remain agnostic of the source of its configuration, whether it be from a file, a network stream, or programmatic construction.

## Lifecycle & Ownership
- **Creation:** Instances are created by a higher-level schema parser or configuration loader when it encounters a corresponding directive in the source data (e.g., a "title" key in a JSON object). It is not intended for direct instantiation by game logic developers.
- **Scope:** The lifecycle of a UIPropertyTitle instance is extremely short. It exists only for the duration of the schema modification transaction. It is created, its modify method is immediately called, and it becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. No explicit cleanup is required due to its simple, stateless nature.

## Internal State & Concurrency
- **State:** Immutable. The internal *title* field is final and is assigned only once during construction. Instances of this class do not change after they are created.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. The same instance can be safely shared and used across multiple threads.
    - **Warning:** While the UIPropertyTitle object itself is thread-safe, the Schema object passed to its modify method is likely not. Synchronization and proper thread management must be handled by the caller responsible for the Schema modification pipeline.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UIPropertyTitle(String) | constructor | O(1) | Constructs a new metadata object with the specified title. |
| modify(Schema) | void | O(1) | Applies the encapsulated title to the target Schema object. |

## Integration Patterns

### Standard Usage
This class is not meant to be used directly. It is part of the internal schema processing machinery. A parser or processor would use it as follows.

```java
// A SchemaProcessor would create and apply this metadata
Schema targetSchema = new Schema();
String titleFromSource = "Player Health";

// The processor instantiates the command
Metadata titleMetadata = new UIPropertyTitle(titleFromSource);

// The command is executed against the target
titleMetadata.modify(targetSchema);
```

### Anti-Patterns (Do NOT do this)
- **Reusing Instances:** Do not cache or hold references to UIPropertyTitle instances. They are lightweight, transient objects that should be created on-demand during configuration parsing.
- **Manual Modification:** Do not attempt to use this class to dynamically update UI titles at runtime. It is designed for initial schema configuration only. Runtime UI updates are handled by a different subsystem.

## Data Pipeline
The primary role of this class is to act as a data carrier in the pipeline that transforms a declarative configuration source into a fully hydrated Schema object.

> Flow:
> Configuration File (e.g., JSON) -> Schema Parser -> **UIPropertyTitle** (Instantiation) -> Schema::modify -> In-Memory Schema State

