---
description: Architectural reference for RelativeIntegerRange
---

# RelativeIntegerRange

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient Value Object

## Definition
```java
// Signature
public class RelativeIntegerRange {
```

## Architecture & Concepts
The RelativeIntegerRange class is a specialized data structure designed to represent a range between two integers, where each boundary can be either an absolute value or a value relative to a given context. It serves as a core component within the server's command system, allowing command arguments to specify ranges like `5..10` (absolute) or `~-2..~+2` (relative).

Architecturally, this class is a composition of two RelativeInteger objects, one for the minimum bound and one for the maximum. Its primary responsibility is not just to store these bounds, but to resolve them against a contextual base value and produce a concrete integer. This resolution logic is the class's main function.

Integration with Hytale's serialization framework is a key design feature. The static CODEC field exposes this class to the engine's data-driven systems, allowing ranges to be defined in configuration files (e.g., for loot tables, mob attribute randomization) and transparently deserialized into a usable object.

## Lifecycle & Ownership
- **Creation:** A RelativeIntegerRange instance is created in one of two primary ways:
    1. **Deserialization:** The Hytale Codec engine instantiates the class via its protected no-argument constructor when parsing data from a configuration file or network packet. The static CODEC field dictates this process.
    2. **Direct Instantiation:** The command argument parsing system or other server logic may instantiate it directly using its public constructors when a range is defined programmatically or parsed from a command string.
- **Scope:** This object is transient and has a very short lifecycle. It typically exists only for the duration of a single operation, such as the parsing and execution of one server command.
- **Destruction:** The object is managed by the Java Garbage Collector and is eligible for cleanup as soon as the operation that created it completes and no references remain.

## Internal State & Concurrency
- **State:** The internal state consists of a `min` and `max` RelativeInteger. While the fields can be mutated during the codec's build process, the object is *effectively immutable* from a consumer's perspective after it has been constructed, as there are no public setters.
- **Thread Safety:** This class is thread-safe. The `getNumberInRange` method is safe to call from multiple threads concurrently. It performs read-only operations on its internal state and utilizes `ThreadLocalRandom` for random number generation, which is specifically designed to avoid contention in multi-threaded server environments. No external locking is required.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNumberInRange(int base) | int | O(1) | Resolves the min/max bounds against the base value and returns a single random integer within that range (inclusive). If min and max resolve to the same value, that value is returned deterministically. |

## Integration Patterns

### Standard Usage
This class is intended to be used by systems that need to resolve a potentially relative numeric range into a concrete value. The most common use case is within command execution logic.

```java
// Assume 'range' is a RelativeIntegerRange parsed from a command argument
// and 'context' provides the base value (e.g., a player's Y-coordinate).
RelativeIntegerRange range = new RelativeIntegerRange(5, 10);
int baseCoordinate = 64;

// Get a random number within the absolute range 5..10
int absoluteResult = range.getNumberInRange(baseCoordinate); // Result is between 5 and 10

// Example with a relative range
RelativeInteger relativeMin = new RelativeInteger(-2, true); // ~-2
RelativeInteger relativeMax = new RelativeInteger(2, true);  // ~+2
RelativeIntegerRange relativeRange = new RelativeIntegerRange(relativeMin, relativeMax);

// Get a random number relative to the base coordinate (62..66)
int relativeResult = relativeRange.getNumberInRange(baseCoordinate);
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Context:** Passing a meaningless or zero base to `getNumberInRange` when the range is known to be relative will produce incorrect results. The `base` parameter is critical for relative calculations.
- **Manual Serialization:** Do not attempt to manually serialize or deserialize this object. The provided static CODEC is the canonical and version-safe mechanism for persistence and networking. Bypassing it can lead to data corruption or incompatibility with future engine updates.

## Data Pipeline
RelativeIntegerRange primarily functions as a data-processing component within a larger system, such as command parsing or configuration loading.

> **Command System Flow:**
> Command String (`/somecommand 10..20`) -> Command Argument Parser -> **RelativeIntegerRange Instance** -> Command Executor -> `getNumberInRange(base)` -> Game World Logic

> **Configuration Flow:**
> JSON/NBT File (`{ "damageRange": { "Min": 5, "Max": 10 } }`) -> Codec Deserializer -> **RelativeIntegerRange Instance** -> Game System (e.g., MobFactory) -> `getNumberInRange(level)` -> Mob Stat Calculation

