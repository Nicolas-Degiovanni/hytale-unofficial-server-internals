---
description: Architectural reference for RelativeInteger
---

# RelativeInteger

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Value Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class RelativeInteger {
```

## Architecture & Concepts
The RelativeInteger class is a foundational component within the server's command parsing and execution system. It encapsulates a numeric value that can be interpreted in two distinct ways: absolute or relative. This pattern is critical for game commands that operate on coordinates or counts, allowing users to specify exact values (e.g., 100) or values relative to a context (e.g., ~10, meaning "10 more than the current value").

This class acts as a strongly-typed, intermediate representation of a command argument. It is created during the parsing stage from a raw string token and is consumed during the command execution stage.

The inclusion of a static **CODEC** field signifies that RelativeInteger is not just an in-memory command argument type, but also a fully serializable data structure. This allows it to be reliably transmitted over the network, saved to disk, or used in any system that relies on the Hytale codec framework for data representation. It bridges the gap between unstructured user input and the engine's structured data model.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created via the static factory method **parse** when the command system processes a user's input string. Alternatively, instances are materialized by the Hytale codec framework during deserialization, which uses the protected no-argument constructor and populates fields via the defined **CODEC**.
- **Scope:** Short-lived and transient. An instance of RelativeInteger typically exists only for the duration of a single command's parse-and-execute cycle. It holds no external resources.
- **Destruction:** As a simple value object, it is managed entirely by the Java Garbage Collector. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The internal state consists of an integer **value** and a boolean **isRelative**. The object is effectively immutable after construction. While the fields are not declared final, no public methods exist to modify the state after instantiation. The protected constructor is for framework use only.
- **Thread Safety:** This class is inherently thread-safe. Its immutable nature ensures that instances can be safely shared and read across multiple threads without synchronization. The static **parse** method is also re-entrant and thread-safe, as it operates solely on its inputs and local variables.

## API Surface
The public API is minimal and focused on parsing, inspection, and resolution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String, ParseResult) | RelativeInteger | O(L) | **Primary Entry Point.** Static factory to create an instance from a string. Handles `~` prefix and integer parsing. Returns null on failure and updates the ParseResult. |
| resolve(int baseValue) | int | O(1) | **Core Logic.** Calculates the final integer. If relative, returns baseValue + rawValue; otherwise, returns rawValue. |
| getRawValue() | int | O(1) | Returns the stored integer value, ignoring its relativity. |
| isRelative() | boolean | O(1) | Returns true if the value was parsed with a `~` prefix. |

## Integration Patterns

### Standard Usage
The intended use is within a command argument parser. The parser extracts a string token, passes it to the **parse** method, and then uses the resulting object to calculate a final value during command execution.

```java
// Context: Inside a command execution handler
String userInput = "~-5";
ParseResult result = new ParseResult();
RelativeInteger relInt = RelativeInteger.parse(userInput, result);

if (relInt == null) {
    // Parsing failed, result contains the error message
    player.sendMessage(result.getFailureMessage());
    return;
}

// Assume player's current score is the base value
int currentScore = player.getScore();
int finalScore = relInt.resolve(currentScore); // Resolves to currentScore - 5

player.setScore(finalScore);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring ParseResult:** The **parse** method communicates failure by returning null. The caller **must** check for a null return and inspect the provided ParseResult object for the user-facing error message. Failure to do so will result in NullPointerExceptions.
- **Direct Instantiation from User Input:** Do not use `new RelativeInteger()` with unvalidated user input. The static **parse** method is the designated, safe pathway that handles string cleaning, error handling, and correct flag setting.
- **Misusing resolve:** Calling `resolve(0)` on a relative integer is a logical error. The purpose of a relative value is to be resolved against a meaningful base context, not an arbitrary default.

## Data Pipeline
RelativeInteger serves as a critical transformation step, converting unstructured text into a structured, computable value.

> Flow:
> Raw Command String -> Command Argument Tokenizer -> **RelativeInteger.parse()** -> **RelativeInteger Instance** -> Command Executor -> **resolve(contextValue)** -> Final Integer Value

