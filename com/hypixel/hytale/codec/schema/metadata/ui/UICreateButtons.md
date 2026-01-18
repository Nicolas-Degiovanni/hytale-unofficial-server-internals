---
description: Architectural reference for UICreateButtons
---

# UICreateButtons

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient

## Definition
```java
// Signature
public class UICreateButtons implements Metadata {
```

## Architecture & Concepts
The **UICreateButtons** class is a specialized Metadata object used within the engine's declarative UI configuration system. It is not a manager or a service, but rather a short-lived data container that represents a single configuration instruction: "define the set of buttons for a specific UI screen".

This class embodies the Command Pattern. An instance of **UICreateButtons** encapsulates all the information needed to perform an action—in this case, populating a **Schema** object with an array of **UIButton** configurations. This approach decouples the source of the UI definition (e.g., a JSON asset) from the object being configured (the **Schema**).

It operates as part of a larger collection of **Metadata** objects that are serially applied to a central **Schema** instance to build up a complete UI or game configuration before it is consumed by rendering and logic systems.

## Lifecycle & Ownership
The lifecycle of a **UICreateButtons** instance is extremely brief and tied to the configuration application process.

- **Creation:** Instantiated programmatically by a higher-level configuration loader or UI builder. This typically occurs when the engine parses a UI definition file and translates its contents into corresponding **Metadata** objects.
- **Scope:** Transactional. The object exists only for the duration of the configuration step. It is created, its **modify** method is called exactly once, and it is then discarded.
- **Destruction:** The object becomes eligible for garbage collection immediately after the **modify** method returns. Holding a reference to it post-modification serves no purpose and is considered a memory leak.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal **buttons** array is a private final field initialized at construction. The class exposes no methods to alter this array after instantiation. This design guarantees that a given **UICreateButtons** instance represents a consistent and predictable configuration instruction.
- **Thread Safety:** Inherently thread-safe. As an immutable data carrier with no side effects beyond its explicit **modify** contract, it can be safely created, passed, and read across multiple threads without synchronization. The **modify** method itself, however, is not guaranteed to be thread-safe, as it mutates the passed-in **Schema** object. Concurrency control is the responsibility of the caller managing the **Schema**.

## API Surface
The public contract is minimal, focusing entirely on its role as a configuration command.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UICreateButtons(UIButton... buttons) | constructor | O(1) | Constructs the metadata object with a specific set of button definitions. |
| modify(Schema schema) | void | O(1) | Applies the button configuration to the target **Schema**. Throws **NullPointerException** if schema is null. |

## Integration Patterns

### Standard Usage
**UICreateButtons** is intended to be used as part of a larger configuration sequence. It is instantiated and immediately used to modify a **Schema** object, typically within a builder or factory context.

```java
// Assume 'schemaBuilder' is an object responsible for constructing a Schema
// and 'buttonDefs' is a pre-existing array of UIButton objects.

Schema targetSchema = schemaBuilder.getSchema();
Metadata buttonMetadata = new UICreateButtons(buttonDefs);

// Apply the configuration command to the schema
buttonMetadata.modify(targetSchema);

// The buttonMetadata object is no longer needed.
```

### Anti-Patterns (Do NOT do this)
- **Reference Hoarding:** Do not store instances of **UICreateButtons** in long-lived collections or as member variables. Its purpose is fulfilled the moment **modify** is called.
- **Schema Re-application:** Do not call the **modify** method on the same instance with different **Schema** objects. While technically possible, it violates the intended transactional nature of the object.
- **Null Schema:** Passing a null **Schema** to the **modify** method will result in a **NullPointerException**. The caller must ensure the target **Schema** is valid.

## Data Pipeline
This class is a component in the engine's configuration data pipeline, translating high-level definitions into a concrete, usable schema.

> Flow:
> UI Asset (JSON/YAML) -> Asset Deserializer -> **UICreateButtons Instance** -> Schema.modify() -> Finalized Schema -> UI Rendering System

