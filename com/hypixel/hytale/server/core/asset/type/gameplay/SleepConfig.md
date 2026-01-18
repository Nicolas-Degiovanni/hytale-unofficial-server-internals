---
description: Architectural reference for SleepConfig
---

# SleepConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Data Object

## Definition
```java
// Signature
public class SleepConfig {
```

## Architecture & Concepts
The SleepConfig class is a passive data object that encapsulates the server-side rules governing the player sleeping mechanic. It is not a service or manager; instead, it serves as a strongly-typed configuration model that decouples game logic from static configuration values.

Its primary architectural feature is the static CODEC field, which integrates the class directly into the server's asset loading and deserialization pipeline. This allows game designers to define sleep behavior—such as when players can sleep and when they wake up—in external configuration files (e.g., JSON). The game server then loads these files at runtime, populating a SleepConfig instance that is consumed by higher-level game systems.

This pattern centralizes sleep-related constants and logic, ensuring that systems like the WorldTimeManager or PlayerActionValidator can operate on a consistent and authoritative set of rules.

### Lifecycle & Ownership
-   **Creation:** An instance of SleepConfig is not meant to be created directly. It is instantiated by the server's asset loading system when it deserializes a corresponding configuration file using the provided static CODEC. A static DEFAULT instance also exists as a fallback, created during class loading.
-   **Scope:** A configured instance persists for the lifetime of the game world it is associated with. It is a long-lived, read-only snapshot of the world's sleep configuration.
-   **Destruction:** The object is marked for garbage collection when the server or world context that holds a reference to it is shut down. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The object is **Effectively Immutable**. While its fields are technically mutable during construction via the CODEC builder, they are not exposed through public setters. Once deserialization is complete, its state is fixed for its entire lifecycle.
-   **Thread Safety:** The class is **Thread-Safe for read operations**. Because its internal state does not change after construction, multiple threads (e.g., game loop, player-specific threads) can safely invoke its methods without external synchronization or locks.

## API Surface
The public API provides methods to query the configured sleep rules.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWakeUpHour() | float | O(1) | Returns the configured in-game hour (e.g., 5.5 for 05:30) when players wake up. |
| getAllowedSleepHoursRange() | double[] | O(1) | Returns a two-element array defining the start and end in-game hours for the sleep window. May be null. |
| isWithinSleepHoursRange(gameTime) | boolean | O(1) | Determines if the provided game time falls within the allowed sleeping window. Correctly handles ranges that cross midnight. |
| computeDurationUntilSleep(now) | Duration | O(1) | Calculates the time remaining until the next sleep window opens. Returns Duration.ZERO if sleeping is always allowed. |
| getSleepStartTime() | LocalTime | O(1) | A convenience method that converts the start of the sleep range into a LocalTime object. Returns null if no range is set. |

## Integration Patterns

### Standard Usage
The SleepConfig object should be retrieved from a central context, such as a World or Server object. Game logic then uses this instance to validate player actions.

```java
// Retrieve the world-specific sleep configuration
SleepConfig sleepRules = currentWorld.getConfiguration(SleepConfig.class);
LocalDateTime gameTime = currentWorld.getTime();

// Check if a player is allowed to sleep right now
if (sleepRules.isWithinSleepHoursRange(gameTime)) {
    player.goToSleep();
} else {
    // Inform the player how long they must wait
    Duration waitTime = sleepRules.computeDurationUntilSleep(gameTime);
    player.sendMessage("You can sleep in " + formatDuration(waitTime));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new SleepConfig()`. This creates an object with default values and completely bypasses the server's intended configuration loaded from assets. Always retrieve the configured instance from the appropriate game context.
-   **State Mutation:** Do not attempt to modify the array returned by getAllowedSleepHoursRange. While the array object itself is mutable, altering it constitutes a mutation of shared state and can lead to unpredictable behavior across the server. The configuration should be treated as strictly read-only.
-   **Relying on the DEFAULT:** Avoid referencing SleepConfig.DEFAULT in game logic. The default instance is a fallback for the asset system and may not reflect the server's actual configuration.

## Data Pipeline
SleepConfig is the terminal point of a configuration loading pipeline. It does not process data itself but rather represents the structured result of that pipeline.

> Flow:
> Configuration File (e.g., world_settings.json) -> Server Asset Loader -> **SleepConfig.CODEC** -> **SleepConfig Instance** -> Game Logic (e.g., WorldTimeManager)

