---
description: Architectural reference for the Axis enum, a core type for network protocol serialization.
---

# Axis

**Package:** com.hypixel.hytale.protocol.packets.buildertools
**Type:** Value Type / Utility

## Definition
```java
// Signature
public enum Axis {
```

## Architecture & Concepts
The Axis enum is a fundamental, type-safe representation of the three primary Cartesian axes: X, Y, and Z. It serves a critical role within the network protocol layer, specifically for packets related to in-game builder tools.

Its primary architectural function is to act as a strict contract between the low-level integer values transmitted over the network and the high-level, readable constants used in game logic. By encapsulating the integer-to-axis mapping, it eliminates the use of "magic numbers" (e.g., 0, 1, 2) in the application code, thereby increasing clarity and reducing the potential for serialization or logic errors.

The inclusion of the static factory method fromValue centralizes the deserialization logic, ensuring that any attempt to create an Axis from an invalid integer value results in a controlled ProtocolException, preventing data corruption further down the processing pipeline.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated once by the JVM during the class-loading phase. They are compile-time constants and exist before any application code is executed. The internal VALUES array is also initialized at this time.
- **Scope:** Application-scoped. An instance of Axis (e.g., Axis.X) is a static singleton that persists for the entire lifetime of the application.
- **Destruction:** The enum constants are reclaimed by the garbage collector only when the application's ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Deeply immutable. Each enum constant holds a private final integer value that is assigned at creation and can never be modified. The static VALUES array is also effectively immutable.
- **Thread Safety:** Inherently thread-safe. Due to its complete immutability, Axis can be safely read, passed, and compared across any number of threads without requiring locks or any other synchronization primitives. This makes it highly suitable for use in concurrent systems, such as network packet processing threads and the main game loop.

## API Surface
The public contract of Axis is minimal and focused on serialization and deserialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the integer representation of the axis for network serialization. |
| fromValue(int value) | Axis | O(1) | Deserializes an integer into an Axis constant. Throws ProtocolException if the value is out of bounds. |
| VALUES | Axis[] | O(1) | A statically cached array of all Axis constants. Prefer this over the implicit values() method to avoid array allocation. |

## Integration Patterns

### Standard Usage
The primary use case involves converting to and from the integer representation during packet serialization and deserialization.

```java
// Deserialization: Reading from a network buffer
int axisValue = buffer.readVarInt();
Axis toolAxis = Axis.fromValue(axisValue);

// Logic: Using the type-safe enum
if (toolAxis == Axis.Y) {
    // Perform vertical building action
}

// Serialization: Writing to a network buffer
Axis currentAxis = Axis.Z;
buffer.writeVarInt(currentAxis.getValue());
```

### Anti-Patterns (Do NOT do this)
- **Using Ordinals:** Never use the built-in `ordinal()` method for serialization. The protocol relies on the explicit integer defined in the constructor. The order of enum declarations could change, breaking network compatibility.
  ```java
  // INCORRECT - Brittle and error-prone
  int value = myAxis.ordinal(); 
  ```
- **Manual Value Checking:** Do not manually check for valid integer ranges before calling fromValue. The method is designed to perform this validation and throw a standardized exception.
  ```java
  // REDUNDANT - The fromValue method already does this
  if (value >= 0 && value <= 2) {
      axis = Axis.fromValue(value);
  }
  ```
- **Comparison by Value:** Avoid comparing enum instances by their underlying integer value. This defeats the purpose of type safety. Always compare the enum constants directly.
  ```java
  // BAD PRACTICE - Less readable and bypasses type safety
  if (myAxis.getValue() == 0) { ... }

  // CORRECT - Type-safe and clear
  if (myAxis == Axis.X) { ... }
  ```

## Data Pipeline
Axis is a data-translation component that sits at the boundary of the network protocol and game logic.

> Flow:
> Network Byte Stream -> Protocol Deserializer reads an integer -> `Axis.fromValue(intValue)` -> **Axis Enum Instance** -> Builder Tool System -> Game Logic

