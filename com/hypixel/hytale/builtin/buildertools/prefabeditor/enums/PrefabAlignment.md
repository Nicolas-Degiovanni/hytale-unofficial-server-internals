---
description: Architectural reference for PrefabAlignment
---

# PrefabAlignment

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.enums
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum PrefabAlignment {
```

## Architecture & Concepts
PrefabAlignment is a type-safe enumeration that defines a constrained set of coordinate system origins for placing and manipulating prefabs within the builder tools. Its primary architectural role is to eliminate "magic string" or integer-based constants, providing compile-time safety and improved code clarity when specifying how a prefab should be oriented relative to a target position.

The enum encapsulates two fundamental alignment strategies:
1.  **ANCHOR:** Aligns the prefab based on its pre-defined anchor point. This is the standard mode for structured placement.
2.  **ZERO:** Aligns the prefab based on its local origin (0,0,0). This is typically used for absolute, coordinate-based positioning.

By design, this class couples the alignment logic with the necessary data for its user interface representation, namely the localization string key. This localizes the concern of UI presentation directly within the data model, simplifying UI code that needs to render dropdowns or tooltips for this setting.

### Lifecycle & Ownership
-   **Creation:** Instances of PrefabAlignment are constructed and initialized by the Java Virtual Machine during class loading. Only two instances, ANCHOR and ZERO, will ever exist.
-   **Scope:** The enum constants are static and persist for the entire lifetime of the application. They are effectively global, immutable singletons.
-   **Destruction:** Resources are reclaimed by the JVM during application shutdown. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** The internal state of each enum constant is **immutable**. The localizationString field is private and final, assigned only once during static initialization by the JVM.
-   **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, instances can be safely read, passed, and compared across any number of threads without locks or other synchronization primitives.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLocalizationString() | String | O(1) | Retrieves the localization key used to fetch the human-readable name for UI elements. |
| values() | PrefabAlignment[] | O(1) | *Implicit Method.* Returns a cached, pre-allocated array of all defined constants (ANCHOR, ZERO). |
| valueOf(String) | PrefabAlignment | O(N) | *Implicit Method.* Resolves a string name to its corresponding enum constant. Throws IllegalArgumentException if no match is found. |

## Integration Patterns

### Standard Usage
This enum is intended to be used as a parameter for configuring builder tools or prefab-related operations. The client code retrieves the localization key to display a user-friendly option, and the system uses the enum instance itself for control flow.

```java
// System logic using the enum for control flow
void applyPrefab(Prefab prefab, PrefabAlignment alignment) {
    Vector3f origin = (alignment == PrefabAlignment.ANCHOR)
        ? prefab.getAnchorPoint()
        : Vector3f.ZERO;
    // ... proceed with placement logic using the calculated origin
}

// UI code using the enum to populate a dropdown
for (PrefabAlignment alignment : PrefabAlignment.values()) {
    String uiText = localization.getString(alignment.getLocalizationString());
    dropdown.addOption(uiText, alignment);
}
```

### Anti-Patterns (Do NOT do this)
-   **Ordinal-Based Logic:** Do not use the `ordinal()` method for serialization or conditional logic. The integer value is fragile and will change if the declaration order of the enum constants is modified, leading to severe and hard-to-diagnose bugs.

    ```java
    // DO NOT DO THIS
    if (alignment.ordinal() == 0) { // Fragile: Breaks if ANCHOR is not first
        // ...
    }
    ```

-   **String Comparison:** Never convert the enum to a string for comparison. Use direct object identity comparison, which is significantly faster and safer.

    ```java
    // DO NOT DO THIS
    if (alignment.toString().equals("ANCHOR")) {
        // ...
    }

    // CORRECT
    if (alignment == PrefabAlignment.ANCHOR) {
        // ...
    }
    ```

## Data Pipeline
PrefabAlignment acts as a data value that flows from the user interface into the core builder tool logic. It does not process data itself but rather directs how other systems process spatial data.

> Flow:
> UI Control (e.g., Dropdown) -> User Selection -> **PrefabAlignment** instance passed to command -> Prefab Placement Service -> World State Modification

