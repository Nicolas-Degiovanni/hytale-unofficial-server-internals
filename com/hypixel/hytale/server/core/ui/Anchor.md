---
description: Architectural reference for Anchor, the core data model for UI element positioning.
---

# Anchor

**Package:** com.hypixel.hytale.server.core.ui
**Type:** Data Model / POJO

## Definition
```java
// Signature
public class Anchor {
```

## Architecture & Concepts

The Anchor class is a fundamental data model within the Hytale UI framework. It does not represent a service or manager, but rather a passive data structure that defines the layout constraints of a single UI element relative to its parent container. Its design is analogous to constraint-based layout systems found in modern UI development, such as CSS box model properties or Auto Layout in iOS.

The most critical architectural feature of the Anchor class is its static **CODEC** field. This signifies that Anchor objects are not intended to be manually instantiated and configured in Java code. Instead, they are designed to be deserialized from data files (e.g., JSON or a similar format) at runtime. This data-driven approach allows UI layouts to be defined, modified, and extended by designers and modders without requiring changes to the core engine source code.

Each field within the Anchor, such as *left*, *width*, or *height*, is of the type Value<Integer>. This wrapper is significant, as it allows the layout system to interpret values with different units, such as absolute pixels, percentages relative to the parent, or other dynamic calculations.

## Lifecycle & Ownership

-   **Creation:** An Anchor instance is created exclusively by the Hytale serialization system (the Codec engine) when a UI definition asset is loaded from disk. The static CODEC field provides the blueprint for this deserialization process, mapping data keys like "Left" or "Width" to the corresponding fields in the object.
-   **Scope:** The lifetime of an Anchor object is strictly tied to the UI component it describes. It is instantiated when the component is loaded into memory and persists for as long as that component exists.
-   **Destruction:** The object is eligible for garbage collection once its owning UI component is destroyed and all references to it are released. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency

-   **State:** The Anchor is a mutable data container. Its state consists of a collection of layout constraints that can be modified after creation via public setters. This mutability is intended for use by the UI layout engine during its calculation and resolution passes.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization mechanisms. All interaction with an Anchor instance, from its creation during asset loading to its use by the layout engine, must be performed on the main UI thread.

    **WARNING:** Accessing or modifying an Anchor instance from any other thread will result in race conditions, leading to unpredictable and non-deterministic UI layout behavior, visual corruption, or application crashes.

## API Surface

The primary public contract is the data structure defined for serialization. The methods are for programmatic modification, which is an edge case.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setLeft(Value<Integer>) | void | O(1) | Sets the left-edge constraint for the UI element. |
| setWidth(Value<Integer>) | void | O(1) | Sets the width constraint for the UI element. |
| setVertical(Value<Integer>) | void | O(1) | A shorthand constraint to center the element vertically. |

## Integration Patterns

### Standard Usage

The standard and intended usage pattern for Anchor is declarative, within a UI asset definition file. A developer or designer defines the anchor properties, and the engine handles the instantiation and integration.

```json
// Example: A UI component definition in a JSON asset
{
  "type": "Panel",
  "anchor": {
    "Left": { "value": 10, "unit": "pixels" },
    "Right": { "value": 10, "unit": "pixels" },
    "Top": { "value": 10, "unit": "pixels" },
    "Height": { "value": 50, "unit": "pixels" }
  },
  "children": [ ... ]
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new Anchor()` in game logic. The UI system is designed to manage the lifecycle of these objects based on data assets. Programmatic creation is reserved for highly specialized dynamic UI generation and should be avoided.
-   **Multi-threaded Modification:** Never modify an Anchor's properties from a background thread. All UI state, including layout constraints, must be managed by the main UI thread to prevent severe rendering and state corruption issues.
-   **Over-constraining:** Defining conflicting constraints, such as *Left*, *Right*, and *Width* simultaneously, can lead to ambiguous layout behavior. The layout engine will have to resolve this ambiguity, but the results may not be what you expect. Define the minimum set of constraints required to achieve the desired layout.

## Data Pipeline

The Anchor class is a data product of the asset loading pipeline and a data source for the UI layout engine.

> Flow:
> UI Asset File (e.g., JSON) -> Asset Loading Service -> **Codec Engine using Anchor.CODEC** -> **Anchor Instance** -> UI Layout Engine -> Final Component Transform (Position & Size)

