---
description: Architectural reference for BotConfig
---

# BotConfig

**Package:** com.hypixel.hytale.server.core.command.commands.debug.stresstest
**Type:** Value Object / DTO

## Definition
```java
// Signature
public class BotConfig {
```

## Architecture & Concepts
BotConfig is an immutable data container designed to encapsulate all configuration parameters for a simulated bot entity within the server's stress-testing framework. It serves as a simple, self-contained manifest that defines a bot's behavior, spawn location, and server-side properties.

Architecturally, this class decouples the stress test command-and-control system from the bot implementation itself. The command handler is responsible for parsing user input or script files to populate a BotConfig instance. This instance is then passed by value to the bot spawning and management systems.

The core design principle is **immutability**. By making all fields final, the configuration for a bot is guaranteed to be constant throughout its lifecycle. This prevents state corruption and race conditions in a highly concurrent environment where thousands of bots may be simulated simultaneously across multiple threads.

## Lifecycle & Ownership
- **Creation:** A BotConfig instance is created by a higher-level system, typically a stress test command handler or a scenario loader, in response to a request to initiate a bot simulation. It is instantiated with all required parameters via its public constructor.
- **Scope:** The object's lifetime is directly tied to the bot or group of bots it configures. It is a transient object, held as a reference by the bot's AI controller or state machine. It persists only as long as the bot exists in the world.
- **Destruction:** The object is eligible for garbage collection once the bot it configures is despawned and all references to the BotConfig instance are dropped. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. All member fields are declared `public final` and are assigned exactly once within the constructor. The state of a BotConfig object cannot be mutated after its creation.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its immutable design, a single BotConfig instance can be safely read by multiple threads without any external synchronization or locking mechanisms. This is a critical feature for the performance and stability of the multi-threaded stress-testing engine.

## API Surface
The public contract of BotConfig consists solely of its constructor and its public final fields. There are no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| radius | double | O(1) | Defines the maximum horizontal distance a bot will wander from its spawn point. |
| flyYHeight | Vector2d | O(1) | A vector representing the minimum (x) and maximum (y) altitude for flying behavior. |
| flySpeed | double | O(1) | The movement speed of the bot when it is in a flying state. |
| spawn | Transform | O(1) | The precise initial world position and orientation for the bot upon spawning. |
| viewRadius | int | O(1) | The server-side view distance (in chunks) to be simulated for this bot. |

## Integration Patterns

### Standard Usage
BotConfig is intended to be created once and passed to the systems responsible for bot management. It acts as a configuration blueprint.

```java
// A stress test command handler creates a configuration for a new wave of bots.
Transform spawnPoint = new Transform(new Vector3d(100, 64, 100), Quaternion.IDENTITY);
Vector2d flightAltitude = new Vector2d(80.0, 120.0);
int serverViewDistance = 8;

BotConfig config = new BotConfig(
    50.0,           // radius
    flightAltitude, // flyYHeight
    12.5,           // flySpeed
    spawnPoint,     // spawn
    serverViewDistance
);

// The configuration object is passed to the core stress test manager.
stressTestManager.spawnBotGroup(50, config);
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify the fields of a BotConfig instance after creation (e.g., via reflection). This violates its immutability contract and can lead to unpredictable and non-reproducible behavior in stress tests. If a different configuration is needed, create a new BotConfig instance.
- **Null Parameters:** Do not pass null for the `spawn` or `flyYHeight` constructor arguments. The class performs no internal null-safety checks, and doing so will cause a NullPointerException in downstream systems that consume the configuration, such as the bot's physics or AI update loops.

## Data Pipeline
BotConfig is not part of a data processing pipeline but rather a configuration flow. It carries initial parameters from the point of definition to the point of execution.

> Flow:
> Server Console Command -> Command Parser -> **BotConfig** (Instantiation) -> StressTestManager -> Bot Spawner -> Individual Bot Entity AI

