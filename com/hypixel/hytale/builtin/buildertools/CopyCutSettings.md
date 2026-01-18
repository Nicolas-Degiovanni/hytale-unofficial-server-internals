---
description: Architectural reference for CopyCutSettings
---

# CopyCutSettings

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Utility

## Definition
```java
// Signature
public class CopyCutSettings {
```

## Architecture & Concepts
CopyCutSettings is a static utility class that defines a set of bitmask flags for configuring clipboard operations within the Hytale builder tools. It is not a service or a component with behavior; rather, it serves as a contract, providing a stable and readable API for specifying which data types should be included in a copy or cut operation.

The core design is a bitfield, where each constant represents a single, unique bit. This allows developers to combine multiple settings into a single integer using bitwise operations. This pattern decouples the clipboard system from the user interface, allowing for complex selection criteria to be passed efficiently to backend systems. For example, a user might select to copy blocks and entities but not fluids; the resulting configuration would be `BLOCKS | ENTITIES`.

This class centralizes the definitions, preventing the use of "magic numbers" throughout the codebase and ensuring that all systems involved in clipboard operations adhere to the same set of rules.

### Lifecycle & Ownership
-   **Creation:** As a class containing only static final constants, CopyCutSettings is never instantiated. Its fields are initialized by the Java Virtual Machine's ClassLoader when the class is first referenced by another part of the engine.
-   **Scope:** The constants defined within this class are globally accessible and persist for the entire lifetime of the application after the class has been loaded.
-   **Destruction:** The class and its static data are unloaded when the application terminates and its ClassLoader is garbage collected.

## Internal State & Concurrency
-   **State:** This class is stateless and immutable. All members are compile-time constants (`public static final int`), and no instance state can ever exist.
-   **Thread Safety:** CopyCutSettings is unconditionally thread-safe. Its constants can be read from any thread at any time without requiring synchronization or locks.

## API Surface
The public contract of this class consists entirely of its constant fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NONE | int (Bitmask) | N/A | Represents no selection. A clipboard operation with this setting will do nothing. |
| CUT | int (Bitmask) | N/A | Flag indicating a cut operation. If present, source data will be removed after copying. |
| EMPTY | int (Bitmask) | N/A | Includes empty air blocks in the selection. |
| BLOCKS | int (Bitmask) | N/A | Includes standard block data in the selection. |
| ENTITIES | int (Bitmask) | N/A | Includes entity data (e.g., NPCs, creatures) in the selection. |
| TINT_MAP | int (Bitmask) | N/A | Includes custom color and tinting data in the selection. |
| KEEP_ANCHORS | int (Bitmask) | N/A | Preserves the original world coordinates (anchors) of the selection. |
| FLUIDS | int (Bitmask) | N/A | Includes fluid block data (e.g., water, lava) in the selection. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to combine multiple flags using the bitwise OR operator to form a complete configuration for a clipboard operation. This integer is then passed to the relevant builder tool system.

```java
// Example: Configure a copy operation to include blocks, entities, and fluids.
int copyConfiguration = CopyCutSettings.BLOCKS | CopyCutSettings.ENTITIES | CopyCutSettings.FLUIDS;

// The resulting integer is passed to the clipboard service.
// WARNING: The BuilderClipboard class is hypothetical for this example.
BuilderClipboard clipboard = context.getService(BuilderClipboard.class);
clipboard.copySelectionToBuffer(selectionArea, copyConfiguration);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The class has no public constructor and is not designed to be instantiated. Attempting `new CopyCutSettings()` will result in a compile-time error.
-   **Using Magic Numbers:** Avoid using the raw integer values directly. Code such as `clipboard.copy(8)` is unreadable and brittle. If the constant value for BLOCKS were to change, this code would break silently. Always use the named constants.
-   **Incorrect Bitwise Operations:** Do not use arithmetic addition (`+`) to combine flags. While it may work for some combinations, it is not semantically correct for bitfields and will produce incorrect results if flags are ever changed. Always use the bitwise OR (`|`) operator.

## Data Pipeline
CopyCutSettings does not process data itself. Instead, its constants act as configuration inputs that direct the flow and transformation of data within the builder tool pipeline.

> Flow:
> Builder Tool UI -> **CopyCutSettings Flags** (Combined via bitwise OR) -> Clipboard Service -> World Serializer (selectively reads blocks, entities, etc., based on flags) -> In-Memory Clipboard Buffer

