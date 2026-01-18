---
description: Architectural reference for OptionArg
---

# OptionArg

**Package:** com.hypixel.hytale.server.core.asset.type.buildertool.config.args
**Type:** Transient Data Model

## Definition
```java
// Signature
public class OptionArg extends ToolArg<String> {
```

## Architecture & Concepts

The OptionArg class is a specialized data model within the server-side Builder Tool framework. Its primary function is to represent a tool argument that must be selected from a predefined list of string-based choices, effectively acting as a dynamic enumeration.

This class serves as a critical bridge between three distinct domains:
1.  **Asset Configuration:** It is designed to be deserialized directly from tool definition files using its static CODEC field. This allows game designers to define tool behaviors, including valid options, in external assets without modifying server code.
2.  **Command Processing:** It encapsulates the validation logic for parsing user input. The fromString method provides a robust mechanism to convert raw string input from a player command into a valid, canonical option.
3.  **Network Protocol:** It is responsible for serializing its state into a specific network packet, BuilderToolOptionArg, for transmission to the client. This enables the client-side user interface to accurately render the available choices for a given tool.

Unlike a simple Java Enum, an OptionArg is defined at runtime from configuration, providing significant flexibility for content creation and modding.

### Lifecycle & Ownership
-   **Creation:** OptionArg instances are almost exclusively created by the Hytale Codec system during server startup or when builder tool assets are hot-reloaded. The system reads a tool definition asset, finds an argument of this type, and uses the static CODEC to instantiate and populate the object. Direct manual instantiation is rare and reserved for programmatic tool generation.
-   **Scope:** The lifetime of an OptionArg is bound to the lifetime of its parent Builder Tool configuration. It persists in memory as long as the tool it belongs to is registered with the server.
-   **Destruction:** The object is marked for garbage collection when its parent tool configuration is unloaded. There are no explicit cleanup or disposal methods.

## Internal State & Concurrency
-   **State:** This class is a **mutable** state container. Its primary state consists of the `options` array, which defines the set of valid choices, and the inherited `value` field, which holds the currently selected choice. While the `options` array is intended to be immutable after creation, it is not enforced by the language.
-   **Thread Safety:** OptionArg is **not thread-safe**. It is a simple data object and contains no internal synchronization mechanisms. It is designed to be configured and loaded within a single thread (typically the main server thread) and subsequently read during command processing, which should also occur in a controlled, single-threaded context per-player or per-world.

**WARNING:** Concurrent access, especially writing to the `value` field or modifying the `options` array from multiple threads, will result in race conditions and undefined behavior. Do not share instances across threads without external locking.

## API Surface

The public API is focused on validation and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromString(String str) | String | O(N) | Parses raw user input. Validates against the internal options list by case-insensitive name or by index. Throws IllegalArgumentException on failure. |
| toOptionArgPacket() | BuilderToolOptionArg | O(1) | Creates a new, specialized network packet containing the current value and the list of all available options. |
| setupPacket(BuilderToolArg packet) | void | O(1) | Configures a generic BuilderToolArg packet, injecting its specific OptionArg data and setting the correct argument type flag. |

## Integration Patterns

### Standard Usage

This class is not typically used directly by developers. It is a component of the Builder Tool system, which orchestrates its lifecycle. The most common interaction pattern is the system invoking fromString to validate player input for a tool command.

```java
// Hypothetical system-level usage
// Assume 'optionArg' was loaded from an asset file
OptionArg optionArg = loadedTool.getArgument("material");

// Player executes a command: /tool set material "wood"
String playerInput = "wood";

try {
    // Validate and canonicalize the input
    String validatedMaterial = optionArg.fromString(playerInput);

    // Apply the validated value
    optionArg.setValue(validatedMaterial);
    // ... update game state
} catch (IllegalArgumentException e) {
    // Send "Invalid material" error to player
}
```

### Anti-Patterns (Do NOT do this)
-   **Post-Creation State Mutation:** Do not modify the `options` array after the object has been created by the codec. The state of the `options` array is considered immutable after deserialization and is sent to clients on that assumption.
-   **Ignoring Validation:** Do not bypass the fromString method and set the internal `value` field directly with unvalidated user input. This would break the contract of the class and could lead to data corruption or server errors.
-   **Shared Mutable Instances:** Do not use a single OptionArg instance to manage state for multiple distinct operations simultaneously. Each argument in a tool configuration should have its own unique instance.

## Data Pipeline

The OptionArg class participates in two primary data flows: configuration loading and network synchronization.

> **Configuration Flow:**
> Builder Tool Asset File (JSON/HOCON) -> Server Asset Loader -> Hytale Codec System -> **OptionArg Instance** (In-Memory)

> **Network & Command Flow:**
> **OptionArg Instance** -> `toOptionArgPacket()` -> Server Network Layer -> Client UI (Displays options)
>
> Player Input (String) -> Server Command Parser -> `**OptionArg.fromString()**` -> Validated Value -> Game State Update

