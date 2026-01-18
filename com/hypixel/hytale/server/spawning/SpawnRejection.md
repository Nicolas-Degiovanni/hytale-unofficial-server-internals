---
description: Architectural reference for SpawnRejection
---

# SpawnRejection

**Package:** com.hypixel.hytale.server.spawning
**Type:** Data Type / Enumeration

## Definition
```java
// Signature
public enum SpawnRejection {
```

## Architecture & Concepts
The SpawnRejection enum provides a type-safe, explicit contract for communicating the reasons for a failed entity spawn attempt within the server's spawning subsystem. It serves as a simple but critical data structure that decouples the spawn validation logic from the calling systems that initiate the spawn.

By using a dedicated enum instead of generic integers or strings, the system avoids "magic values" and enhances code clarity. Any component interacting with the spawning pipeline can deterministically check for specific failure modes using a `switch` statement or equality checks, leading to more robust and maintainable error handling. This enum is a terminal state in a spawn validation check; it represents the *result* of a process, not the process itself.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine (JVM) during class loading. The static `VALUES` array is also initialized at this time. This process is automatic and occurs once when the class is first referenced.
- **Scope:** As static constants, all instances of SpawnRejection (`OUTSIDE_LIGHT_RANGE`, `INVALID_POSITION`, etc.) persist for the entire lifetime of the server application. They are effectively global, immutable singletons.
- **Destruction:** The enum and its constants are garbage collected only when the server's class loader is unloaded, which typically happens during a full server shutdown. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** SpawnRejection is **immutable**. The enum constants themselves have no mutable fields. The public `VALUES` array is a static final reference, and while the contents of an array are technically mutable, modifying it would violate its design contract and is strongly discouraged.
- **Thread Safety:** This class is inherently **thread-safe**. Its immutable nature guarantees that it can be safely accessed and passed between any number of threads without requiring locks or other synchronization primitives.

## API Surface
The primary API consists of the enum constants themselves and the pre-cached array of all possible values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OUTSIDE_LIGHT_RANGE | SpawnRejection | O(1) | Constant representing a spawn failure due to insufficient or excessive light levels. |
| INVALID_SPAWN_BLOCK | SpawnRejection | O(1) | Constant representing a failure because the target block is not a valid spawn surface. |
| INVALID_POSITION | SpawnRejection | O(1) | Constant for failures where the position is geometrically invalid (e.g., inside a wall). |
| NO_POSITION | SpawnRejection | O(1) | Constant indicating that no suitable spawn position could be found in the search area. |
| NOT_BREATHABLE | SpawnRejection | O(1) | Constant for failures where the spawn location is in a medium the entity cannot breathe (e.g., underwater for a terrestrial mob). |
| OTHER | SpawnRejection | O(1) | A generic catch-all constant for unspecified or miscellaneous spawn failures. |
| VALUES | SpawnRejection[] | O(1) | A static, cached array of all defined enum constants. Provides a minor performance benefit over calling `values()` repeatedly. |

## Integration Patterns

### Standard Usage
SpawnRejection is intended to be used as a return type for methods that validate potential spawn locations. The caller then inspects the result to handle the failure appropriately.

```java
// A spawner system checks for a valid position
SpawnRejection reason = spawnValidator.checkPosition(world, targetPos);

if (reason != null) { // A non-null value indicates failure
    log.warn("Spawn failed at " + targetPos + " due to: " + reason.name());
    // Potentially retry or log metrics based on the specific reason
} else {
    // Proceed with spawning the entity
}
```

### Anti-Patterns (Do NOT do this)
- **Returning Null:** A validation method returning a SpawnRejection should never return null to indicate success. The absence of a rejection is success. A null return value is ambiguous and error-prone.
- **Using Ordinals:** Do not use the `ordinal()` method for serialization or conditional logic. If a new constant is added or the order is changed, all dependent logic will break. Use the `name()` method for a stable identifier.
- **Extensibility:** Enums cannot be extended. If a new rejection reason is required, it must be added directly to the SpawnRejection source file. Do not attempt to create a parallel system of constants.

## Data Pipeline
SpawnRejection acts as a terminal data object in the spawn validation pipeline. It does not process data itself but rather represents the final output of the validation stage.

> Flow:
> Spawner System -> Request Spawn Location -> Spawn Validation Logic -> **SpawnRejection** (Result) -> Spawner System (Handles Result)

