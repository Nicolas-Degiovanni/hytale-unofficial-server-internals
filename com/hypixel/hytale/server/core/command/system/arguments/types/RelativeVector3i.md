---
description: Architectural reference for RelativeVector3i
---

# RelativeVector3i

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class RelativeVector3i {
```

## Architecture & Concepts
The RelativeVector3i class is a fundamental data structure within the server-side command system. It is not a service or manager, but rather a specialized value object designed to represent a three-dimensional coordinate that can be either absolute or relative to a reference point.

Its primary role is to encapsulate coordinate arguments from commands like `/teleport` or `/setblock`. This allows users to specify positions relative to their current location (e.g., `~10 5 ~-2`) or as absolute world coordinates (e.g., `100 64 250`). This functionality is achieved through composition, where the class holds three instances of RelativeInteger, one for each axis (X, Y, Z).

A critical feature is the static CODEC field. This integrates the class directly into Hytale's serialization framework, allowing it to be encoded and decoded from network packets or configuration files automatically. This makes it a transparent data type for any system that uses the Hytale codec engine.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the server's command parsing engine. When a command containing coordinate-like arguments is processed, the parser instantiates a RelativeVector3i to represent those arguments. It may also be created during deserialization from a data source via its static CODEC.
- **Scope:** Short-lived and transactional. An instance typically exists only for the duration of a single command's execution. It is created, used to calculate a final position, and then immediately becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. No manual resource cleanup is necessary.

## Internal State & Concurrency
- **State:** The object's state is **mutable** upon creation. The internal fields for x, y, and z are not final and are assigned directly by the constructor or the builder-based CODEC. However, after instantiation, it is treated as an immutable value object by the command system. Modifying its state post-creation is considered an anti-pattern.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization. It is designed to be created, processed, and discarded within the scope of a single thread, typically the server's main tick or command processing thread.

**WARNING:** Do not share instances of RelativeVector3i across threads or cache them in a concurrent environment without implementing external synchronization. Doing so will lead to unpredictable behavior.

## API Surface
The public API is minimal, focusing entirely on resolving the relative coordinate into an absolute one.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolve(Vector3i base) | Vector3i | O(1) | Resolves the relative vector against a base Vector3i, returning a new absolute Vector3i. |
| resolve(int x, int y, int z) | Vector3i | O(1) | Resolves the relative vector against base integer components. This is the core logic. |
| isRelativeX() | boolean | O(1) | Returns true if the X component is relative (e.g., `~10`). |
| isRelativeY() | boolean | O(1) | Returns true if the Y component is relative. |
| isRelativeZ() | boolean | O(1) | Returns true if the Z component is relative. |

## Integration Patterns

### Standard Usage
The intended use is within a command handler to calculate a target world position based on the command sender's location.

```java
// Inside a command execution context
// 'sender' is the entity that executed the command
// 'relativePos' is the RelativeVector3i parsed from command arguments

Vector3i senderPosition = sender.getPosition();
Vector3i targetPosition = relativePos.resolve(senderPosition);

// Use the absolute targetPosition to perform a game world action
world.teleportEntity(sender, targetPosition);
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not attempt to modify the internal RelativeInteger objects after the RelativeVector3i has been created. It should be treated as immutable.
- **Long-Term Storage:** Do not cache or store instances of this class. They are lightweight and should be created on-demand by the command parser.
- **Cross-Thread Usage:** Never pass a RelativeVector3i to another thread for resolution. The resolution should occur on the same thread that handles the command execution to avoid race conditions with the base entity's position.

## Data Pipeline
RelativeVector3i acts as an intermediate representation between raw user input and a final, absolute world coordinate.

> Flow:
> Raw Command String (`/tp ~5 ~ ~10`) -> Command Argument Parser -> **RelativeVector3i** -> Command Executor -> `resolve(base)` -> Absolute `Vector3i` -> World State Change

