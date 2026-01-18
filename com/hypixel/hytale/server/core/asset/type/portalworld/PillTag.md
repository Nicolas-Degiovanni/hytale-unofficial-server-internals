---
description: Architectural reference for PillTag
---

# PillTag

**Package:** com.hypixel.hytale.server.core.asset.type.portalworld
**Type:** Transient Data Object

## Definition
```java
// Signature
public class PillTag {
```

## Architecture & Concepts
The PillTag class is a simple, immutable data container that represents a descriptive label within the user interface, typically associated with a server or world listing. Its primary architectural role is to serve as a deserialization target for data defined in game asset files.

This class is not a service or manager; it is a value object. Its structure is defined by the static **CODEC** field, which dictates how asset data (e.g., from a JSON file) is mapped into a live Java object. This tight coupling with the Hytale serialization framework makes it a fundamental building block for loading and representing configuration data.

A PillTag encapsulates two core properties required for rendering:
1.  A **translationKey**, which is a string identifier used by the localization system to fetch the display text in the user's language.
2.  A **color**, which defines the background or text color of the rendered UI element.

The presence of the getMessage method indicates that this is a "smart" data object, capable of transforming its internal state into a Message object, the standard type consumed by higher-level UI and localization systems.

## Lifecycle & Ownership
-   **Creation:** PillTag instances are created almost exclusively by the Hytale **Codec** system during the asset loading phase. They are instantiated and populated when a parent asset, such as a portal world definition, is parsed from disk. Manual instantiation is strongly discouraged as it bypasses the intended data pipeline.
-   **Scope:** The lifetime of a PillTag is strictly bound to its containing asset. It persists in memory only as long as the parent object that owns it is referenced. It is a short-lived object with no global state.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for cleanup as soon as their parent asset is unloaded or goes out of scope. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The object's state (translationKey, color) is populated once during deserialization. While the fields are not declared final, there are no public setters, making the object **effectively immutable** after construction. All public methods are pure-read operations.
-   **Thread Safety:** Due to its immutable nature post-construction, PillTag is inherently **thread-safe**. An instance can be safely read, shared, and passed between multiple threads without requiring any external synchronization or locks.

## API Surface
The public API is minimal, focusing entirely on providing read-only access to the tag's properties.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTranslationKey() | String | O(1) | Retrieves the raw localization key for the tag's text. |
| getMessage() | Message | O(1) | Constructs and returns a new Message object, ready for consumption by the localization engine. |
| getColor() | Color | O(1) | Retrieves the Color object associated with this tag for rendering. |

## Integration Patterns

### Standard Usage
The intended use is to receive PillTag objects from a parent data structure (like a world definition) and use their properties to configure UI components. Developers should never create PillTag instances directly.

```java
// Example: A system receives a PortalWorld asset and renders its tags
PortalWorldDefinition worldDef = assetManager.get(portalAssetKey);

for (PillTag tag : worldDef.getPillTags()) {
    // Use the tag's data to configure a UI element
    Message tagText = tag.getMessage();
    Color tagColor = tag.getColor();
    uiBuilder.createTagComponent(tagText, tagColor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PillTag()`. The resulting object will be uninitialized and will cause NullPointerExceptions when its methods are called. All instances must be created via the asset loading and codec pipeline.
-   **State Modification:** Do not attempt to modify the internal state of a PillTag using reflection. The rest of the engine assumes these objects are immutable, and such changes can lead to inconsistent and unpredictable UI rendering.

## Data Pipeline
The PillTag is a product of the asset deserialization pipeline. It represents a transformation of static configuration data into a usable in-memory object.

> Flow:
> Game Asset File (e.g., JSON) -> Asset Loading Service -> **BuilderCodec Deserializer** -> **PillTag Instance** -> Game Logic / UI System -> Rendered UI Element

