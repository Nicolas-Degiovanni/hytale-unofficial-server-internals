---
description: Architectural reference for PatchStyle
---

# PatchStyle

**Package:** com.hypixel.hytale.server.core.ui
**Type:** Data Object / POJO

## Definition
```java
// Signature
public class PatchStyle {
```

## Architecture & Concepts

The PatchStyle class is a fundamental data object within the Hytale UI framework, specifically designed to represent the visual properties of a 9-patch image. A 9-patch image is a stretchable bitmap that can be used as a scalable background for UI components, ensuring that corners are not distorted when the component is resized.

This class is not a service or a manager; it is a pure data container. Its primary role is to act as a schema, defined in Java, for styling information that is typically declared in external UI definition files (e.g., JSON or HOCON). The engine's UI asset loading system uses the static **CODEC** field to deserialize these definitions into a hydrated PatchStyle Java object at runtime.

A key architectural feature is the use of the **Value** generic type for all properties. This abstraction layer allows style properties to be defined not just as static literals, but as references to theme variables or other dynamic sources. This is critical for supporting features like UI theming and dynamic style updates without hardcoding values.

## Lifecycle & Ownership

-   **Creation:** PatchStyle instances are overwhelmingly created by the UI asset pipeline during the deserialization of UI layout files. The static **BuilderCodec** is the factory responsible for this instantiation. Manual construction via the *new* keyword is reserved for procedurally generated UI that does not originate from a static asset file.
-   **Scope:** The lifetime of a PatchStyle instance is tightly coupled to the UI component that owns it. It is a transient object, not a shared global resource. If a UI screen is created, its components will hold references to their respective PatchStyle objects.
-   **Destruction:** When a UI component is destroyed and removed from the render tree (e.g., a screen is closed), its associated PatchStyle instance becomes eligible for garbage collection. There are no manual cleanup or disposal methods.

## Internal State & Concurrency

-   **State:** The class is fully mutable. It is designed as a simple data structure whose fields are populated either by the **CODEC** during asset loading or through the public builder-style setter methods. It does not cache any data or maintain connections to external systems.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms. All mutation and access should occur on the main client thread. Modifying a PatchStyle instance from a background thread while it is actively being used by the rendering system will result in undefined behavior and visual artifacts.

## API Surface

The public API consists of a standard builder pattern for programmatic configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setTexturePath(Value) | PatchStyle | O(1) | Sets the asset path for the source texture. |
| setBorder(Value) | PatchStyle | O(1) | Sets a uniform border size for all four sides of the 9-patch. |
| setHorizontalBorder(Value) | PatchStyle | O(1) | Overrides the uniform border for the left and right sides. |
| setVerticalBorder(Value) | PatchStyle | O(1) | Overrides the uniform border for the top and bottom sides. |
| setColor(Value) | PatchStyle | O(1) | Applies a color tint to the texture. The value is typically a hex string. |
| setArea(Value) | PatchStyle | O(1) | Defines a specific sub-region of the source texture to use. |

## Integration Patterns

### Standard Usage

While PatchStyle is typically defined declaratively in asset files, it can be constructed programmatically for dynamic UI elements. The builder pattern is the correct way to do this.

```java
// Example of creating a style for a dynamically generated button
PatchStyle dynamicButtonStyle = new PatchStyle()
    .setTexturePath(Value.of("hytale:ui/textures/button_default"))
    .setBorder(Value.of(8))
    .setColor(Value.of("#A0C4FF"));

// This style object would then be passed to a UI component's constructor
// or applyStyle method.
```

### Anti-Patterns (Do NOT do this)

-   **Stateful Reuse:** Do not share a single PatchStyle instance across multiple, independent UI components. If one component's style is later modified, it will unintentionally affect all other components sharing that instance. Treat them as disposable value objects.
-   **Cross-Thread Modification:** Never modify a PatchStyle object from a worker thread if that object is part of an active UI component tree. All UI state mutations must be marshaled back to the main thread.

## Data Pipeline

The primary flow for PatchStyle involves its creation from static data files and its consumption by the UI rendering engine.

> Flow:
> UI Definition File (e.g., screen.json) -> Hytale Asset Loader -> **BuilderCodec** -> **PatchStyle Instance** -> UI Component State -> UI Renderer -> GPU Texture Draw Call

