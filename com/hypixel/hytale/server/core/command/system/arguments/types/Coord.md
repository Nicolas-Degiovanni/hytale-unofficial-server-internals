---
description: Architectural reference for Coord
---

# Coord

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class Coord {
```

## Architecture & Concepts
The Coord class is a specialized value object responsible for parsing and resolving coordinate arguments within the server's command system. It serves as an intermediate representation, converting raw string inputs from command issuers into absolute, usable world coordinates.

Its primary architectural function is to decouple command execution logic from the complexities of coordinate notation. The system supports several notations:
*   **Absolute:** A standard number (e.g., `100`).
*   **Relative:** A number prefixed with a tilde (`~`), indicating an offset from a base coordinate (e.g., `~10`).
*   **Chunk-based:** A number prefixed with a `c`, representing a value in chunk units, which is then converted to block units (e.g., `c2` becomes `64`).
*   **Height-based:** A number prefixed with an underscore (`_`), used for the Y-axis to find the highest solid block at a given X/Z location, plus an optional offset (e.g., `_5`).

By encapsulating this parsing and resolution logic, command implementations can operate on simple double values without needing to be aware of the original syntax used by the player or system. This promotes cleaner, more maintainable command code.

### Lifecycle & Ownership
-   **Creation:** A Coord instance is created exclusively through the static factory method **parse**. This occurs within the command argument parsing pipeline when a coordinate argument type is expected.
-   **Scope:** The object's lifetime is extremely brief and is tied to the execution of a single command. It is created, used to resolve a coordinate, and then immediately becomes eligible for garbage collection.
-   **Destruction:** The object is managed by the JVM's garbage collector. As no long-term references are ever held, it is typically reclaimed very quickly after the command completes.

## Internal State & Concurrency
-   **State:** The Coord class is **Immutable**. All internal fields (value, height, relative, chunk) are final and are set only once during construction. This design guarantees that a parsed coordinate's meaning cannot change after creation.
-   **Thread Safety:** This class is inherently **Thread-Safe**. Its immutability ensures that it can be safely shared and read by multiple threads without any risk of data corruption or race conditions. No synchronization mechanisms are required.

## API Surface
The public API is focused on parsing and resolution. Simple boolean getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String str) | static Coord | O(N) | **Primary entry point.** Parses a string into a new Coord instance. N is the string length, which is trivially small. |
| resolveXZ(double base) | double | O(1) | Resolves the coordinate for a horizontal (X or Z) axis. The base value is used for relative (`~`) calculations. |
| resolveYAtWorldCoords(double base, World world, double x, double z) | double | O(1)* | Resolves the coordinate for the vertical (Y) axis. If the height flag (`_`) is set, this method performs a world lookup to find the terrain height. Throws GeneralCommandException if the required chunk is not loaded. *Complexity depends on chunk lookup performance.* |

## Integration Patterns

### Standard Usage
The class is intended to be used by the command framework. A command argument parser will invoke **parse** on the raw string input, and the command's execution logic will then use a resolver method to get the final coordinate.

```java
// Context: Inside a command executor
// senderLocation is the origin point for relative coordinates
String rawX = commandArgs.nextString(); // e.g., "~10"
String rawY = commandArgs.nextString(); // e.g., "_"
String rawZ = commandArgs.nextString(); // e.g., "c4"

try {
    Coord coordX = Coord.parse(rawX);
    Coord coordZ = Coord.parse(rawZ);

    // Resolve X and Z first, as they may be needed for Y-height resolution
    double finalX = coordX.resolveXZ(senderLocation.getX());
    double finalZ = coordZ.resolveXZ(senderLocation.getZ());

    Coord coordY = Coord.parse(rawY);
    double finalY = coordY.resolveYAtWorldCoords(senderLocation.getY(), world, finalX, finalZ);

    // Use finalX, finalY, finalZ for game logic (e.g., teleportation)
} catch (GeneralCommandException e) {
    // Handle parsing or chunk loading failures
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new Coord(...)`. The static **parse** method is the designated entry point and contains the necessary logic to interpret string notation. Bypassing it defeats the purpose of the class.
-   **Incorrect Resolver Usage:** Do not use **resolveXZ** to resolve a Y-coordinate that was parsed with the height flag (`_`). Doing so will ignore the world lookup for terrain height and will likely produce incorrect vertical positioning. Always use **resolveYAtWorldCoords** for the Y-axis.
-   **Ignoring Exceptions:** The **resolveYAtWorldCoords** method can throw a GeneralCommandException if a chunk is not loaded. This is a recoverable error that must be caught and handled gracefully, typically by informing the command issuer that the target location is not accessible.

## Data Pipeline
The flow of data through this component is linear and stateless, transforming a string into a final numeric coordinate.

> Flow:
> Raw Command String -> Command Argument Splitter -> **Coord.parse()** -> **Coord Instance** -> Command Executor -> **coord.resolve...()** -> Final `double` Coordinate

