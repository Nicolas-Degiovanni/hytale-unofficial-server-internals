---
description: Architectural reference for RelativeFloat
---

# RelativeFloat

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class RelativeFloat {
```

## Architecture & Concepts
The RelativeFloat class is a specialized Value Object used within the server's command parsing framework. It encapsulates a floating-point number that can be either **absolute** (e.g., `100.5`) or **relative** (e.g., `~-10`). Its primary architectural role is to decouple the parsing of a command argument from its final application in the game world.

This allows command logic to be written generically, without needing to know if a user's input was relative or absolute. The class handles the parsing of the tilde character (`~`) and provides a single `resolve` method to compute the final value against a given base.

The inclusion of a static `CODEC` field signifies that this class is also a well-defined data structure within the engine's serialization system. This enables relative values to be persisted in configuration files, transmitted over the network, or used in other systems that rely on the Hytale `Codec` framework.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the static `parse` method, which is invoked by the command system's argument parser when processing player or system input. They can also be instantiated during deserialization via the `CODEC`.
- **Scope:** A RelativeFloat is a short-lived object. Its lifetime is typically bound to a single command execution. It is created, used to calculate a final coordinate or value, and then becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or `close` methods.

## Internal State & Concurrency
- **State:** The object's state is defined by two private fields: `value` (a float) and `isRelative` (a boolean). While the class is technically mutable for the `BuilderCodec`, it is **effectively immutable** after construction. Once an instance is created via its public constructor or the `parse` method, its state does not change.
- **Thread Safety:** The class is inherently thread-safe. Since its state is immutable post-construction, a single instance can be safely read by multiple threads without synchronization. The static `parse` factory method is also thread-safe as it operates only on its local arguments and produces a new, independent instance.

## API Surface
The public API provides methods for parsing, resolving, and inspecting the value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String, ParseResult) | RelativeFloat | O(L) | **Static Factory.** Parses a string input. Handles the `~` prefix and empty inputs. Returns null on failure and updates the provided ParseResult. |
| resolve(float baseValue) | float | O(1) | Computes the final value. If relative, returns `baseValue + rawValue`. If absolute, returns `rawValue`. This is the primary method for consumption. |
| getRawValue() | float | O(1) | Returns the stored float value, ignoring whether it is relative or absolute. |
| isRelative() | boolean | O(1) | Returns true if the value was parsed with a `~` prefix. |

## Integration Patterns

### Standard Usage
The intended use is to parse a string argument from a command, check for success, and then resolve the value against a relevant base, such as an entity's current coordinate.

```java
// Within a command execution context
ParseResult result = new ParseResult();
String inputX = "~10.5"; // From command arguments

RelativeFloat relX = RelativeFloat.parse(inputX, result);

if (result.isSuccess()) {
    float currentX = player.getPosition().getX();
    float targetX = relX.resolve(currentX);
    
    // Use targetX to update player position or other game logic
    player.teleportTo(targetX, ...);
} else {
    // Report the parsing failure to the command sender
    player.sendMessage(result.getErrorMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring ParseResult:** The `parse` method returns null upon failure. Code that does not check the state of the `ParseResult` object or for a null return value is susceptible to `NullPointerException`.
- **Misinterpreting Raw Value:** Directly using `getRawValue` without first checking `isRelative` bypasses the core logic of the class. This leads to incorrect calculations, as a relative value of `10.0` will be treated as an absolute coordinate. Always prefer the `resolve` method.

## Data Pipeline
RelativeFloat acts as a key transformation step in the server's command processing pipeline.

> Flow:
> Raw Command String -> Argument Tokenizer -> **RelativeFloat.parse()** -> Command Executor -> **RelativeFloat.resolve(base)** -> Game World State Change

