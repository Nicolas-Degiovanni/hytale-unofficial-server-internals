---
description: Architectural reference for BrushOperationSetting
---

# BrushOperationSetting<T>

**Package:** com.hypixel.hytale.builtin.buildertools.scriptedbrushes.operations.system
**Type:** Transient Data Model

## Definition
```java
// Signature
public class BrushOperationSetting<T> {
```

## Architecture & Concepts
The BrushOperationSetting class is a fundamental data model within the Scripted Brushes framework. It is not a service or a manager, but rather a self-contained descriptor that defines and manages a single, configurable parameter for a BrushOperation. Each instance represents one setting, such as *size*, *material*, or *intensity*.

Architecturally, this class serves as a bridge between raw user input and the type-safe logic of a brush. Its most critical feature is the integration with the server's command argument parsing system via the ArgumentType<T> dependency. This design choice allows brush settings to leverage the same robust and extensible parsing engine used for server commands, ensuring a consistent user experience and reducing code duplication.

A BrushOperationSetting encapsulates the entire schema for a parameter:
*   **Identity:** A unique name and user-facing description.
*   **State:** A default value, a current value, and the last raw input string.
*   **Behavior:** The logic for parsing string input, validating the result, and formatting the value for display.

This encapsulation allows UI systems and other tools to introspect a BrushOperation and dynamically generate controls and validation feedback without needing to understand the operation's internal logic.

## Lifecycle & Ownership
- **Creation:** An instance is created directly by a parent BrushOperation, typically within its constructor. Each BrushOperation is responsible for defining the complete set of BrushOperationSetting objects that it supports.
- **Scope:** The lifecycle of a BrushOperationSetting is strictly bound to its owning BrushOperation instance. It persists as long as its parent operation exists.
- **Destruction:** The object is marked for garbage collection when its parent BrushOperation is destroyed. It holds no external resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is highly **mutable**. Its primary purpose is to hold the runtime state of a setting, specifically the current *value*. The fields *value* and *input* are expected to change frequently as a user configures and uses a brush tool.

- **Thread Safety:** BrushOperationSetting is **not thread-safe**. It is designed with the expectation that it will be created, modified, and read from a single, controlling thread, such as the main game thread or a dedicated world-editing thread. Unsynchronized, concurrent access, especially to methods like setValue or parseAndSetValue, will result in race conditions and undefined behavior. All state transitions must be externally synchronized if multi-threaded access is required.

## API Surface
The public API provides methods for updating the setting's value from either raw input or a pre-validated object, and for retrieving its current state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parseAndSetValue(String[] input) | ParseResult | O(N) | Parses raw string input using the configured ArgumentType. This is the primary method for updating state from user commands or UI fields. The result object must be checked for failure. |
| setValue(T value) | BrushOperationSetting<T> | O(1) | Directly sets the internal value with a type-safe object. Bypasses parsing and validation. |
| setValueUnsafe(String input, Object value) | BrushOperationSetting<T> | O(1) | **Warning:** Unsafe operation that bypasses type checking via a cast. Used for internal state hydration where the type is already guaranteed. Avoid in general application code. |
| getValue() | T | O(1) | Returns the current, strongly-typed value of the setting. May be null if not yet set or if parsing failed. |
| getValueString() | String | O(1) | Returns a user-friendly string representation of the current value, using a custom formatter if one was provided. |

## Integration Patterns

### Standard Usage
A BrushOperation defines its settings as final fields and initializes them in its constructor. The operation's core logic then retrieves the current values from these setting objects when it is executed.

```java
// Example from within a hypothetical "SphereBrushOperation" class

// Settings are defined as instance variables
private final BrushOperationSetting<Integer> sizeSetting;
private final BrushOperationSetting<BlockMaterial> materialSetting;

public SphereBrushOperation() {
    // Settings are instantiated in the constructor
    this.sizeSetting = new BrushOperationSetting<>(
        "size",
        "The radius of the sphere in blocks",
        5, // Default value
        ArgumentTypes.INTEGER, // Use the engine's integer parser
        Validators.range(1, 64) // Add a validator
    );

    this.materialSetting = new BrushOperationSetting<>(
        "material",
        "The block material to place",
        BlockMaterials.STONE, // Default value
        ArgumentTypes.BLOCK_MATERIAL // Use the engine's material parser
    );
}

public void execute(World world, Vector3 position) {
    // The operation's logic retrieves the current values when needed
    int currentSize = this.sizeSetting.getValue();
    BlockMaterial currentMaterial = this.materialSetting.getValue();

    // ... proceed to modify the world using these values
}
```

### Anti-Patterns (Do NOT do this)
- **State Sharing:** Do not share a single BrushOperationSetting instance across multiple BrushOperation instances. Each setting object holds the state for *one* specific operation instance. Reusing them will cause state corruption.
- **Cross-Thread Modification:** Do not call `setValue` or `parseAndSetValue` from a different thread than the one that owns the parent BrushOperation. This will lead to inconsistent state and visibility problems. All modifications should be marshaled to the owner's thread.
- **Ignoring ParseResult:** The ParseResult from `parseAndSetValue` is critical. It indicates success or failure. Failure to check this result means the application will be unaware of invalid user input, and the setting's internal value will remain unchanged, leading to confusing behavior.

## Data Pipeline
This class is a central component in the data flow that translates user intentions from text or UI into type-safe, validated data for use by world-editing tools.

> Flow:
> User Input (UI Text Field / Chat Command) -> Raw String -> **BrushOperationSetting.parseAndSetValue()** -> Internal Typed *value* Field -> BrushOperation.execute() Logic

