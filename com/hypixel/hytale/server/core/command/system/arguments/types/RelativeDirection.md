---
description: Architectural reference for RelativeDirection
---

# RelativeDirection

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Utility Enum

## Definition
```java
// Signature
public enum RelativeDirection {
```

## Architecture & Concepts
The RelativeDirection enum is a foundational component of the server's command system, responsible for translating human-readable directional input into concrete mathematical vectors and axes within the game world. It serves as a critical bridge between natural language commands issued by a player (e.g., "forward", "up", "l") and the server's internal coordinate and physics systems.

Architecturally, this enum encapsulates two distinct responsibilities:
1.  **Parsing & Validation:** Through its static ARGUMENT_TYPE field, it provides a self-contained parser that conforms to the server's command argument framework. It handles string-to-enum conversion, including aliases (e.g., "f" for FORWARD) and error reporting.
2.  **Contextual Transformation:** It provides static utility methods to convert a semantic direction into a mathematical one. This transformation is not absolute; it is critically dependent on the context of the executing entity, specifically their HeadRotation. "Forward" is not a fixed vector but is calculated relative to where an entity is looking.

This design decouples the command definition layer from the complexities of vector math and entity state, allowing command implementers to work with high-level, self-explanatory concepts.

### Lifecycle & Ownership
-   **Creation:** As a Java enum, all instances (FORWARD, BACKWARD, etc.) are constructed and initialized by the JVM during class loading. The static ARGUMENT_TYPE instance is also created at this time. This process is automatic and occurs once when the class is first referenced.
-   **Scope:** Application-wide. The enum constants and the associated ARGUMENT_TYPE parser persist for the entire lifetime of the server process.
-   **Destruction:** The enum and its instances are unloaded by the JVM only when the server application shuts down. There is no manual memory management required.

## Internal State & Concurrency
-   **State:** Inherently immutable. The enum constants are compile-time constants. The class contains no mutable instance or static fields, making its behavior deterministic.
-   **Thread Safety:** This class is unconditionally thread-safe. All methods are pure functions that operate solely on their inputs to produce an output. They do not modify any shared state and can be invoked concurrently from any thread without synchronization.

## API Surface
The public contract is exposed entirely through static members.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ARGUMENT_TYPE | SingleArgumentType | O(1) | The primary integration point for the command system. Provides parsing logic for converting a string into a RelativeDirection. |
| toDirectionVector(direction, headRotation) | static Vector3i | O(1) | **Core Function.** Translates a RelativeDirection into a world-space direction vector, relative to the provided HeadRotation. |
| toAxis(direction, headRotation) | static Axis | O(1) | Determines the primary world axis (X, Y, or Z) that corresponds to the given direction and HeadRotation. |

## Integration Patterns

### Standard Usage
The primary integration path is through the command system. A command's argument signature will declare a parameter of this type, and the framework will use ARGUMENT_TYPE to parse the user's input. The resulting enum value is then used with the entity's current rotation to perform game logic.

```java
// Example of resolving a direction within a command handler
public void execute(CommandContext context) {
    RelativeDirection direction = context.getArgument("direction", RelativeDirection.class);
    HeadRotation playerRotation = context.getSource().getEntity().get(HeadRotation.class);

    // Calculate the actual world vector based on player's view
    Vector3i worldVector = RelativeDirection.toDirectionVector(direction, playerRotation);

    // Use the vector for game logic, e.g., teleportation or block placement
    teleportPlayer(context.getSource(), worldVector);
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Context:** Never call toDirectionVector or toAxis with a default or null HeadRotation. The result will be meaningless or cause a NullPointerException. The entire purpose of this class is to resolve directions *relative* to an entity's orientation.
-   **Manual Parsing:** Do not write your own string-parsing logic (e.g., `if (input.equals("forward"))`). Always use the provided ARGUMENT_TYPE, as it correctly handles all aliases and localization concerns.
-   **Misusing Vectors:** The vectors returned by toDirectionVector are normalized direction vectors. They are not positions. Do not use them directly for setting an entity's world coordinates without first scaling them or adding them to an origin point.

## Data Pipeline
This enum functions as a transformation stage within the server's command processing pipeline.

> Flow:
> Raw Command String -> Command Dispatcher -> **RelativeDirection.ARGUMENT_TYPE.parse()** -> RelativeDirection Enum Instance -> **toDirectionVector()** -> Vector3i -> Game Logic System

