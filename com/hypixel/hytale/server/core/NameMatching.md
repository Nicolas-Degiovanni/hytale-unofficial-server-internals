---
description: Architectural reference for NameMatching
---

# NameMatching

**Package:** com.hypixel.hytale.server.core
**Type:** Utility

## Definition
```java
// Signature
public enum NameMatching {
```

## Architecture & Concepts
The NameMatching enum implements a **Strategy Pattern** to provide a set of predefined, type-safe algorithms for searching collections of objects by a string identifier, such as a player name. It is a foundational utility within the server core, designed to centralize and standardize how partial or case-insensitive names are resolved to specific game objects.

This system decouples the *act of searching* from the specific *method of searching*. Instead of scattering custom string comparison logic throughout the codebase (e.g., for command parsing, target selection, or admin tools), other systems can simply select the desired NameMatching strategy (e.g., EXACT, STARTS_WITH_IGNORE_CASE) and delegate the lookup operation.

The core of its design is a scoring mechanism. Each strategy defines a `Comparator` that assigns a score to potential matches. This allows for "best-fit" resolution rather than simple boolean matching. For example, when searching for "Alex" using the STARTS_WITH strategy, "AlexW" would score higher (be a better match) than "AlexanderTheGreat". An immediate-exit path is provided for exact matches to optimize performance.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances (EXACT, STARTS_WITH, etc.) are instantiated by the JVM during class loading. They are compile-time constants, not objects created at runtime.
- **Scope:** Application-wide. The enum constants persist for the entire lifetime of the server process.
- **Destruction:** The instances are garbage collected only when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** The NameMatching enum and its instances are **deeply immutable**. The internal `comparator` and `equality` fields are final and point to stateless lambda expressions. The class itself holds no mutable state.
- **Thread Safety:** This class is inherently thread-safe. The `find` methods are pure functions that operate solely on their inputs without causing side effects. They can be safely invoked from multiple threads concurrently without any need for external synchronization or locks.

## API Surface
The primary contract is the `find` method, which resolves a string value to an object within a collection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| find(players, value, getter) | T | O(N) | Finds the best-matching object in a collection using the instance's specific matching strategy. Returns null if no suitable match is found. |
| find(players, value, getter, comparator, equality) | T | O(N) | Static variant that allows for a fully custom, one-off search without using a predefined enum strategy. |
| getComparator() | Comparator | O(1) | Returns the scoring comparator associated with the strategy. |

## Integration Patterns

### Standard Usage
The most common use case is resolving a player name from a command argument against a collection of online players. Always use a predefined enum constant.

```java
// How a developer should normally use this
Collection<Player> onlinePlayers = server.getOnlinePlayers();
String inputName = "pla"; // From a command argument

// Use the server's default matching strategy
Player found = NameMatching.DEFAULT.find(onlinePlayers, inputName, Player::getName);

if (found != null) {
    // Logic for the resolved player
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Re-implementation:** Do not write custom loops with `startsWith` or `equalsIgnoreCase` for name lookups. This creates inconsistent matching behavior across the server and bypasses the optimized scoring logic of this utility.
- **Misusing the Static `find`:** Avoid calling the static `find` method with custom comparators unless you are defining a truly unique search behavior that cannot be encapsulated in a new enum constant. For all standard lookups, use the instance method on a predefined strategy (e.g., `NameMatching.EXACT.find(...)`).

## Data Pipeline
NameMatching acts as a resolution step in a data flow, transforming an ambiguous user input string into a concrete object reference.

> Flow:
> User Input (e.g., Command Argument) -> Command Parsing System -> **NameMatching.find()** -> Resolved Game Object (e.g., Player) -> Game Logic Execution

