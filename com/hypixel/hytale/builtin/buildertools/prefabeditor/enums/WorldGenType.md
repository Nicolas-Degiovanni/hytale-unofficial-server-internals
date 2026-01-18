---
description: Architectural reference for WorldGenType
---

# WorldGenType

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor.enums
**Type:** Utility

## Definition
```java
// Signature
public enum WorldGenType {
```

## Architecture & Concepts
The WorldGenType enum provides a type-safe, compile-time constant representation for world generation algorithms available within the Prefab Editor. Its primary architectural function is to decouple the user interface and command systems from the underlying world generation logic.

By representing generation types as enum constants (FLAT, VOID), the system avoids the use of "magic strings" or integers, which are error-prone and difficult to maintain. Each constant encapsulates a localization key, allowing the user-facing name of the generation type to be translated and modified independently of the code that consumes it. This is a critical pattern for internationalization (i18n) and maintaining clean, readable control flow, such as in switch statements.

This enum exists within the *Builder Tools* feature set and is specifically consumed by the Prefab Editor's UI and command handlers to configure a temporary world for asset editing.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. The instances FLAT and VOID are created once and exist as static, singleton objects.
- **Scope:** Application-wide. These constants persist for the entire lifetime of the application.
- **Destruction:** The enum and its constants are unloaded and garbage collected by the JVM only when the application shuts down.

## Internal State & Concurrency
- **State:** WorldGenType is **immutable**. Its internal state, the localizationString, is a final field initialized at creation and cannot be changed.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutability and the JVM's guarantees for enum initialization, it can be safely read and passed between any number of threads without requiring external synchronization or locks.

## API Surface
The public contract is minimal, focused on retrieving the data associated with each constant.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLocalizationString() | String | O(1) | Returns the localization key for the UI. Never returns null. |

## Integration Patterns

### Standard Usage
The primary use case is to pass a specific generation type to a service or to retrieve its display key for a UI component.

```java
// Retrieving the enum from a string (e.g., from a command argument)
WorldGenType type = WorldGenType.valueOf("FLAT");

// Using the enum to control logic
if (type == WorldGenType.FLAT) {
    worldGenerator.generateFlatWorld();
}

// Getting the display key for a UI label
String uiKey = type.getLocalizationString();
uiLabel.setText(Localization.get(uiKey));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Enums cannot be instantiated with the new keyword. Attempting to do so will result in a compile-time error.
- **Comparison by Ordinal:** Do not use the ordinal() method for control flow. If the order of constants in the enum definition changes, the logic will break silently. Always compare instances directly using ==.
- **Hard-coding Localization Keys:** Do not use the raw string "server.commands.editprefab.ui.worldGenType.flat" in UI code. Always retrieve it via the getLocalizationString method to respect the single source of truth.

## Data Pipeline
WorldGenType acts as a data model or a configuration token, not a processing component. It carries information between different layers of the application.

> **UI to Logic Flow:**
> User Selection in UI -> UI Event maps to **WorldGenType.FLAT** -> Command Payload -> World Generation Service -> `switch(type)` -> Algorithm Execution

> **Logic to UI Flow:**
> **WorldGenType.FLAT** -> `getLocalizationString()` -> Localization Service -> "Flat" (Rendered Text)

