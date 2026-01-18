---
description: Architectural reference for SpawnTestResult
---

# SpawnTestResult

**Package:** com.hypixel.hytale.server.spawning
**Type:** Enumeration

## Definition
```java
// Signature
public enum SpawnTestResult {
```

## Architecture & Concepts
SpawnTestResult is a type-safe enumeration that represents the outcome of a server-side entity spawn validation check. It serves as a status code, providing a clear and descriptive result from the complex process of determining if a specific world location is suitable for an entity to spawn.

This enum is a critical component of the server's spawning subsystem. Instead of returning a simple boolean or an ambiguous integer, methods that test spawn locations return an instance of SpawnTestResult. This design pattern forces the calling code to handle various failure scenarios explicitly, leading to more robust and predictable spawning behavior. It effectively decouples the *validation* of a spawn point from the *execution* of the spawn itself.

The primary consumer of this enum is the SpawnSystem, which uses the result to decide whether to proceed with creating an entity, attempt to find a new location, or abandon the spawn attempt entirely.

## Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) when the SpawnTestResult class is loaded. They are static, final objects and cannot be created by application code.
- **Scope:** Application-wide. These constants exist for the entire lifetime of the server process.
- **Destruction:** The constants are garbage collected when the server process terminates and the class loader is unloaded. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant is defined at compile time and cannot be altered at runtime.
- **Thread Safety:** Fully thread-safe. As immutable constants, instances of SpawnTestResult can be safely accessed and passed between any number of threads without requiring synchronization.

## API Surface
The public API consists of the set of defined enumeration constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| TEST_OK | SpawnTestResult | N/A | Indicates the spawn location is valid and spawning can proceed. |
| FAIL_NO_POSITION | SpawnTestResult | N/A | The validation failed because no valid position could be found in the target area. |
| FAIL_INVALID_POSITION | SpawnTestResult | N/A | The provided position is fundamentally invalid (e.g., outside world bounds). |
| FAIL_INTERSECT_ENTITY | SpawnTestResult | N/A | The spawn location's bounding box intersects with an existing entity. |
| FAIL_NO_MOTION_CONTROLLERS | SpawnTestResult | N/A | The entity to be spawned lacks required motion controllers for the target medium (e.g., swimming). |
| FAIL_NOT_SPAWNABLE | SpawnTestResult | N/A | The block or region at the target location is marked as non-spawnable. |
| FAIL_NOT_BREATHABLE | SpawnTestResult | N/A | The entity cannot breathe at the target location (e.g., a land creature spawning underwater). |

## Integration Patterns

### Standard Usage
The intended use is to check the return value from a spawn test method and branch the logic accordingly, typically with a switch statement or if-else chain.

```java
// Standard pattern for handling a spawn test
SpawnTestResult result = spawnSystem.validateSpawnLocation(targetPosition, entityType);

switch (result) {
    case TEST_OK:
        // Proceed with entity creation
        world.spawnEntity(entity, targetPosition);
        break;
    case FAIL_INTERSECT_ENTITY:
    case FAIL_NOT_SPAWNABLE:
        // Log the failure and attempt to find a new nearby position
        log.warn("Spawn failed at " + targetPosition + ", reason: " + result);
        tryAlternativeLocation();
        break;
    default:
        // Handle other critical failures
        log.error("Unhandled spawn failure: " + result);
        abortSpawn();
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not use the ordinal() method for logical comparisons (e.g., `if (result.ordinal() > 0)`). The declaration order of enum constants can change between versions, which would break such logic. Always compare instances directly.
- **String Comparison:** Do not compare the result using its string representation (e.g., `result.toString().equals("TEST_OK")`). This is inefficient and less safe than direct object comparison. Use `result == SpawnTestResult.TEST_OK`.
- **Ignoring Failures:** Code that only checks for `TEST_OK` and ignores all other failure cases is brittle. This can lead to silent failures in the spawning system.

## Data Pipeline
SpawnTestResult is a terminal data object; it is the *output* of a validation pipeline, not a component that transforms data.

> Flow:
> Spawn Request -> World Position Query -> Spawn Location Validator -> **SpawnTestResult** -> Spawning Logic / Retry Logic

