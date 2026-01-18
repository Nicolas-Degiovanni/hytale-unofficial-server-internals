---
description: Architectural reference for PrefabStackingAxis
---

# PrefabStackingAxis

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.enums
**Type:** Enum

## Definition
```java
// Signature
public enum PrefabStackingAxis {
```

## Architecture & Concepts
PrefabStackingAxis is a type-safe enumeration that defines the valid axes for stacking operations within the Prefab Editor build tool. It represents a constrained choice between the horizontal X and Z axes, abstracting away the underlying world coordinate system.

The primary architectural role of this enum is to enforce data integrity and improve code clarity. By using PrefabStackingAxis instead of primitive types like integers or strings (e.g., 0 for X, 1 for Z), the system prevents invalid state and runtime errors. It serves as a contract for any system that performs sequential placement or duplication of prefabs, ensuring that operations are confined to a valid, planar axis.

## Lifecycle & Ownership
- **Creation:** Enum instances are constructed automatically by the Java Virtual Machine (JVM) when the PrefabStackingAxis class is loaded. The constants X and Z are instantiated once and exist as static, final fields.
- **Scope:** These instances are singletons that persist for the entire lifetime of the application. Their lifecycle is tied directly to the class loader.
- **Destruction:** The enum instances are garbage collected only when the defining class loader is unloaded, which typically occurs during application shutdown. Manual destruction is not possible.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant is defined at compile time and cannot be altered.
- **Thread Safety:** PrefabStackingAxis is unconditionally thread-safe. As immutable singletons, its constants can be safely accessed and passed between any number of threads without synchronization. This is a core guarantee of the Java enum type.

## API Surface
The API consists solely of the predefined constants. Standard Java enum methods like values() and valueOf(String) are implicitly available but are part of the language specification, not a custom API for this type.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| X | PrefabStackingAxis | O(1) | Represents the world's X-axis for stacking operations. |
| Z | PrefabStackingAxis | O(1) | Represents the world's Z-axis for stacking operations. |

## Integration Patterns

### Standard Usage
This enum is intended to be used as a parameter in methods that perform directional logic or to be stored as part of a configuration state for a tool.

```java
// Example: Storing the current stacking mode in a tool's state object
ToolState state = new ToolState();
state.setStackingAxis(PrefabStackingAxis.X);

// Example: Using the enum to control logic flow
public void stackPrefab(Prefab prefab, PrefabStackingAxis axis) {
    switch (axis) {
        case X:
            // ... logic to place next prefab along the X-axis
            break;
        case Z:
            // ... logic to place next prefab along the Z-axis
            break;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Do not rely on the enum's ordinal() method for serialization or conditional logic. If the order of declaration (X, Z) changes, all dependent logic will break silently.
- **Null Checks:** Functions accepting a PrefabStackingAxis should be designed to expect a non-null value. Allowing nulls defeats the purpose of type safety and forces defensive null-checking throughout the call stack.

## Data Pipeline
PrefabStackingAxis is not a processing component but rather a data value that flows through other systems. It typically originates from a user interface element and is passed down to the core building logic.

> Flow:
> UI Toggle Click -> SetToolStateCommand (with **PrefabStackingAxis.X**) -> PrefabEditorService -> PrefabPlacementSystem -> World Manipulation API

