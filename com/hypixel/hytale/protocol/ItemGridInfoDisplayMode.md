---
description: Architectural reference for ItemGridInfoDisplayMode
---

# ItemGridInfoDisplayMode

**Package:** com.hypixel.hytale.protocol
**Type:** Enum Constant

## Definition
```java
// Signature
public enum ItemGridInfoDisplayMode {
```

## Architecture & Concepts
The ItemGridInfoDisplayMode enum defines a constrained, type-safe set of identifiers for UI display behaviors. It is a critical component of the network protocol, acting as a data contract between the client and server for how item-related information should be rendered in a grid-based interface, such as an inventory or crafting table.

Its primary architectural function is to replace ambiguous "magic numbers" (e.g., 0, 1, 2) with named constants (Tooltip, Adjacent, None). This improves code readability and maintainability while ensuring that only valid values can be transmitted or processed. The inclusion of explicit integer values and a deserialization method, fromValue, indicates its direct role in the serialization and deserialization pipeline for network packets or save game data.

## Lifecycle & Ownership
- **Creation:** Enum instances are constants, created and initialized by the JVM during class loading. They are not instantiated at runtime by application code. The static VALUES array is also initialized at this time.
- **Scope:** As static constants, all instances of ItemGridInfoDisplayMode exist for the entire lifetime of the application. They are globally accessible and effectively act as singletons for each defined mode.
- **Destruction:** The enum and its instances are unloaded from memory only when the application's ClassLoader is garbage collected, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** ItemGridInfoDisplayMode is **immutable**. Each enum constant holds a final integer value that is assigned at compile time and cannot be changed. The static VALUES array is also effectively immutable after its initial population by the JVM.
- **Thread Safety:** This class is unconditionally **thread-safe**. As an immutable type with no mutable state, its instances can be safely accessed and shared across any number of threads without synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value associated with the enum constant. This is the canonical method for serialization. |
| fromValue(int value) | ItemGridInfoDisplayMode | O(1) | **Static.** Deserializes an integer into its corresponding enum constant. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
This enum is primarily used during the deserialization of network packets or game state data. The fromValue method is the designated factory for converting a raw integer from a data stream into a type-safe object.

```java
// Reading a display mode from a network buffer
int modeId = buffer.readVarInt();
ItemGridInfoDisplayMode displayMode = ItemGridInfoDisplayMode.fromValue(modeId);

// Using the mode in game logic
switch (displayMode) {
    case Tooltip:
        // Show item info in a tooltip
        break;
    case Adjacent:
        // Show item info in an adjacent UI panel
        break;
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals for Serialization:** Do not use the built-in `ordinal()` method for serialization. The integer value of an ordinal can change if the declaration order of the enum constants is modified, which would break network compatibility with older clients or servers. Always use `getValue()`.
- **Ignoring ProtocolException:** The `fromValue` method can throw an exception if it receives an undefined integer. Failure to handle this exception can lead to a client or server crash when processing a malformed or malicious packet.

## Data Pipeline
ItemGridInfoDisplayMode serves as a data model within a larger serialization pipeline. It does not process data itself but represents a state that is converted to and from a raw format.

> **Serialization Flow:**
> Game State (e.g., Container Configuration) -> **ItemGridInfoDisplayMode.Tooltip** -> `getValue()` -> Integer (0) -> Network Buffer

> **Deserialization Flow:**
> Network Buffer -> Integer (0) -> `fromValue(0)` -> **ItemGridInfoDisplayMode.Tooltip** -> UI System Update

