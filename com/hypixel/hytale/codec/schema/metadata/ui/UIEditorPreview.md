---
description: Architectural reference for UIEditorPreview
---

# UIEditorPreview

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient

## Definition
```java
// Signature
public class UIEditorPreview implements Metadata {
```

## Architecture & Concepts
The UIEditorPreview class is a specialized metadata object that functions as a configuration command within the Hytale asset system. Its sole responsibility is to declare how an asset should be previewed within the Hytale UI Editor.

This class embodies the Command Pattern. Rather than having disparate parts of the UI editor directly manipulate a central Schema object, they instead create an instance of UIEditorPreview. This instance encapsulates the *request* to change the preview type. This object is then passed to a central schema processor which executes the modification. This design decouples the UI interaction logic from the core schema mutation logic, promoting cleaner separation of concerns and ensuring that all schema modifications are applied through a controlled, consistent interface (the Metadata interface).

It is not a data container in the traditional sense; it is an instruction that is applied once and then discarded.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by UI event handlers within the Hytale Editor. For example, when a user selects a preview mode from a dropdown menu, the corresponding event listener will construct a new UIEditorPreview with the selected PreviewType.
-   **Scope:** Extremely short-lived. An instance typically exists only for the duration of a single schema modification operation. It is a value object, not a managed service.
-   **Destruction:** Eligible for garbage collection immediately after its `modify` method has been invoked. No long-term references to these objects should be held.

## Internal State & Concurrency
-   **State:** Immutable. The internal `previewType` field is final and is set exclusively at construction time. The object's state cannot be changed after it has been created.
-   **Thread Safety:** Fully thread-safe. Due to its immutable nature, an instance of UIEditorPreview can be safely shared and read across multiple threads without synchronization. However, the Schema object it modifies is likely not thread-safe, and modifications should be synchronized or confined to a single thread.

## API Surface
The public contract is minimal, focusing entirely on its role as a schema modifier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UIEditorPreview(PreviewType) | constructor | O(1) | Constructs a new metadata command with the specified preview type. |
| modify(Schema) | void | O(1) | Applies the encapsulated preview type to the target Schema object. This is the primary execution method. |

## Integration Patterns

### Standard Usage
The intended use is to create an instance and immediately apply it to a Schema object, typically as part of a larger collection of Metadata modifications.

```java
// Assume 'activeSchema' is the Schema object for the asset being edited.
// Assume 'userSelection' is the PreviewType chosen from the UI.

// 1. Create the metadata command based on user input.
UIEditorPreview previewMetadata = new UIEditorPreview(userSelection);

// 2. Apply the command to the schema.
// This might be done directly or via a processing service.
previewMetadata.modify(activeSchema);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Reuse:** Do not retain and reuse instances of UIEditorPreview. They are cheap to create and represent a single, atomic configuration change. Create a new instance for every modification.
-   **External Modification:** Do not attempt to use reflection or other means to change the internal `previewType` after construction. The object's immutability is fundamental to its design.
-   **Manual Schema Access:** Avoid bypassing this class to set the preview type directly on the Schema. Using the Metadata interface ensures that the modification is part of the engine's standardized configuration pipeline.

## Data Pipeline
This class acts as a carrier for a configuration instruction. The "data" is the enum value that flows from user input into the core asset schema.

> Flow:
> User Input in UI Editor -> Event Handler -> **new UIEditorPreview(type)** -> MetadataProcessor.apply() -> Schema.getHytale().setUiEditorPreview() -> Modified Schema State

