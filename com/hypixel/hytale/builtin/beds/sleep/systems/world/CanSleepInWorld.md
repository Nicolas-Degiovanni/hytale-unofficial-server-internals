---
description: Architectural reference for CanSleepInWorld
---

# CanSleepInWorld

**Package:** com.hypixel.hytale.builtin.beds.sleep.systems.world
**Type:** Utility

## Definition
```java
// Signature
public final class CanSleepInWorld {
```

## Architecture & Concepts
CanSleepInWorld is a stateless, server-side domain validator. Its sole responsibility is to encapsulate the business logic that determines if the conditions within a given World are currently suitable for a player to initiate sleep.

This class acts as a centralized predicate function within the sleep gameplay module. Instead of scattering time checks and configuration lookups across multiple systems, this utility provides a single, authoritative source of truth. It decouples the systems that *trigger* sleep (e.g., player interaction with a bed) from the core rules *governing* sleep, promoting cleaner, more maintainable game logic.

It operates by querying two key pieces of world state:
1.  The current in-game time, retrieved from the WorldTimeResource.
2.  The server-defined sleep configuration, retrieved from the SleepConfig asset.

The result is returned as a sealed interface, Result, which provides a strongly-typed outcome, preventing the use of ambiguous booleans or magic numbers.

## Lifecycle & Ownership
As a final class with only a private constructor and a single public static method, CanSleepInWorld has no instance lifecycle.

-   **Creation:** This class is never instantiated. It cannot be created with the new keyword.
-   **Scope:** Its static methods are available globally for the duration of the server's runtime, once the class is loaded by the JVM.
-   **Destruction:** The class is unloaded by the JVM during server shutdown. There is no manual cleanup or destruction process.

## Internal State & Concurrency
-   **State:** CanSleepInWorld is completely stateless. It holds no member variables and does not cache any data between calls. All necessary information is provided via the World parameter in the check method.
-   **Thread Safety:** The check method is inherently thread-safe. It performs read-only operations on the provided World object. However, callers are responsible for ensuring that the World object itself is not being mutated by another thread concurrently in a way that would lead to inconsistent reads. The method itself introduces no race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| check(World world) | static Result | O(1) | Evaluates the provided World to determine if sleep is permissible. Returns a Result object indicating the outcome. |

## Integration Patterns

### Standard Usage
This utility should be used as a precondition check within any system that initiates the sleep process for a player or entity. The system queries this utility first and only proceeds if the result is not negative.

```java
// A hypothetical system checking if a player can sleep.
World currentWorld = ...;
CanSleepInWorld.Result result = CanSleepInWorld.check(currentWorld);

if (result.isNegative()) {
    // Deny sleep and potentially send a reason to the player.
    // Example: if (result instanceof CanSleepInWorld.NotDuringSleepHoursRange r) { ... }
    return;
}

// Proceed with sleep logic...
```

### Anti-Patterns (Do NOT do this)
-   **Re-implementing Logic:** Do not manually check the world time against the sleep configuration in other parts of the codebase. This creates duplicate logic and violates the single source of truth principle provided by this class.
-   **Ignoring the Result Type:** Do not simply check `result == CanSleepInWorld.Status.CAN_SLEEP`. The sealed Result interface provides more specific negative outcomes (e.g., NotDuringSleepHoursRange) that can be used to give players more informative feedback.

## Data Pipeline

CanSleepInWorld does not participate in a data transformation pipeline. Instead, it functions as a query gateway to world state.

> **Query Flow:**
>
> Game System (e.g., BedInteractionSystem) -> **CanSleepInWorld.check(world)**
>
> 1.  **Reads:** WorldConfig for game time pause status.
> 2.  **Reads:** WorldTimeResource for current game time.
> 3.  **Reads:** SleepConfig for valid sleep hour range.
>
> -> Returns Result (CAN_SLEEP, GAME_TIME_PAUSED, or NotDuringSleepHoursRange) -> Game System acts on Result.

