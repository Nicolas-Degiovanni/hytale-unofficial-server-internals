---
description: Architectural reference for MouseButtonState
---

# MouseButtonState

**Package:** com.hypixel.hytale.protocol
**Type:** Enum Constant

## Definition
```java
// Signature
public enum MouseButtonState {
```

## Architecture & Concepts
MouseButtonState is a foundational type-safe enumeration within the Hytale protocol layer. Its primary architectural role is to eliminate "magic numbers" (e.g., 0 for pressed, 1 for released) when serializing and deserializing user input data, particularly from network packets or local input devices.

By providing a constrained set of named constants (Pressed, Released), it enforces a strict contract for any system handling mouse input. This prevents a significant class of bugs related to invalid state values. The static factory method, fromValue, serves as the designated deserialization gateway, ensuring that any integer read from a stream is validated before being converted into a valid state object. Failure to map to a known state results in a deterministic ProtocolException, preventing corrupted data from propagating further into the game logic.

## Lifecycle & Ownership
-   **Creation:** Enum constants are instantiated once by the Java Virtual Machine (JVM) when the MouseButtonState class is loaded. They are compile-time constants, not runtime objects in the traditional sense.
-   **Scope:** As static final instances, they persist for the entire application lifetime. They are effectively global, immutable singletons.
-   **Destruction:** The constants are garbage collected only when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
-   **State:** Inherently immutable. The internal integer value associated with each constant is final and assigned at class-loading time. The state of a MouseButtonState instance can never be changed.
-   **Thread Safety:** This class is unconditionally thread-safe. Due to its immutable and static nature, instances can be safely read, passed, and compared across any number of threads without requiring locks or any other synchronization primitives.

## API Surface
The public contract is minimal, focusing exclusively on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the raw integer value for serialization. |
| fromValue(int value) | MouseButtonState | O(1) | Deserializes an integer into a MouseButtonState instance. Throws ProtocolException if the value is out of bounds. |

## Integration Patterns

### Standard Usage
The fromValue method is the canonical entry point when decoding data from a network buffer or input stream. The resulting enum should then be used for control flow via switch statements or direct comparison.

```java
// Example: Deserializing a mouse event from a network buffer
int rawButtonState = networkBuffer.readVarInt();
MouseButtonState state = MouseButtonState.fromValue(rawButtonState);

// Example: Using the state in game logic
if (state == MouseButtonState.Pressed) {
    player.getCharacter().swingWeapon();
}
```

### Anti-Patterns (Do NOT do this)
-   **Using Raw Integers:** Avoid passing the integer value around. Do not write code like `if (state.getValue() == 0)`. The purpose of the enum is to provide type safety and readability; use direct comparison `state == MouseButtonState.Pressed` instead.
-   **Invalid Instantiation:** Never attempt to create instances of an enum using reflection. This violates fundamental JVM guarantees and will lead to unpredictable behavior or runtime failures.
-   **Ignoring Exceptions:** The ProtocolException thrown by fromValue is a critical signal of data corruption or a protocol mismatch. It must not be caught and ignored, as this would allow an invalid state to be masked.

## Data Pipeline
MouseButtonState acts as a data model, primarily used to translate raw data into a structured, type-safe representation for the rest of the engine.

> Flow:
> Raw Integer (from Network Buffer / Input System) -> **MouseButtonState.fromValue()** -> Game Logic (e.g., InputManager, PlayerController) -> State-driven Actions

