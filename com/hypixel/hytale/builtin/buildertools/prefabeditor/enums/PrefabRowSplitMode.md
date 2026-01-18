---
description: Architectural reference for PrefabRowSplitMode
---

# PrefabRowSplitMode

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.enums
**Type:** Type-Safe Enum

## Definition
```java
// Signature
public enum PrefabRowSplitMode {
```

## Architecture & Concepts
PrefabRowSplitMode provides a compile-time safe, fixed set of strategies for organizing UI elements within the Prefab Editor. It is a fundamental component for defining user-configurable layout behaviors in the builder tools.

This enum decouples the internal logical strategy (e.g., BY_ALL_SUBFOLDERS) from its user-facing representation (a localization string). This pattern is critical for the UI system, ensuring that state controllers and rendering logic operate on well-defined, immutable constants rather than error-prone raw strings or magic numbers. It serves as a data contract between the UI configuration logic, the underlying data model, and the localization system.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. This process is guaranteed by the Java Language Specification to be thread-safe and to occur only once per class loader.
- **Scope:** Instances are static and persist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** Instances are reclaimed by the JVM during application shutdown when the corresponding class loader is garbage collected. Manual destruction is neither possible nor necessary.

## Internal State & Concurrency
- **State:** Inherently immutable. The internal state, the localizationString, is a final field assigned at creation time. Once the enum is loaded by the JVM, its state cannot be modified.
- **Thread Safety:** Fully thread-safe. As immutable singletons, instances of PrefabRowSplitMode can be safely accessed, passed between, and read from any number of threads without requiring any synchronization mechanisms.

## API Surface
The public contract is minimal, focusing on retrieving the associated localization data. It also inherits standard, powerful methods from the base java.lang.Enum class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLocalizationString() | String | O(1) | Retrieves the localization key associated with the enum constant. This key is used by the LocalizationManager to fetch the user-facing display text. |
| values() | PrefabRowSplitMode[] | O(N) | **Inherited.** Returns a new array containing all defined enum constants in their declaration order. Primarily used for populating UI elements like dropdowns. |
| valueOf(String name) | PrefabRowSplitMode | O(N) | **Inherited.** Returns the enum constant with the specified name. Throws IllegalArgumentException if no such constant exists. Used for deserialization. |

## Integration Patterns

### Standard Usage
This enum is typically used to populate UI selection components or to drive conditional logic within the Prefab Editor's view controller.

```java
// Example: Populating a UI dropdown with localized options
for (PrefabRowSplitMode mode : PrefabRowSplitMode.values()) {
    String uiLabel = localizationManager.get(mode.getLocalizationString());
    uiDropdown.addOption(uiLabel, mode);
}

// Example: Using in a switch statement to control behavior
switch (currentUserSelection) {
    case BY_ALL_SUBFOLDERS:
        // Implement logic to recursively split rows by folder structure
        break;
    case BY_SPECIFIED_FOLDER:
        // Implement logic to split rows based on a user-provided folder
        break;
    case NONE:
    default:
        // Render rows in a flat list
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Reliance on Ordinals:** Never use the `ordinal()` method for serialization or persistent storage. The integer value is fragile and will change if the enum declaration order is modified, leading to critical data corruption on subsequent application loads.
- **String Comparison:** Do not use `toString()` for logical comparisons. The `toString()` method is intended for debugging and its format is not guaranteed. Always compare enum instances directly using either the `==` operator or the `equals()` method.

## Data Pipeline
As a data definition class, PrefabRowSplitMode does not process data itself but serves as a static data point within a larger flow.

> Flow:
> User Interaction (e.g., Dropdown Selection) -> UI Event -> **PrefabRowSplitMode** (as event payload) -> Prefab Editor State Controller -> UI Rendering Logic

