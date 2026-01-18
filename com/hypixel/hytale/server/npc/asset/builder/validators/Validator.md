---
description: Architectural reference for the Validator abstract base class.
---

# Validator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public abstract class Validator {
}
```

## Architecture & Concepts
The Validator class is an abstract base class that defines the fundamental contract for all validation logic within the server-side NPC asset build pipeline. It establishes a common interface for a series of checks that are executed against raw NPC asset data before it is compiled into a final, game-ready format.

This class is a key component of a Strategy design pattern. The core system, likely an NpcAssetBuilder, is configured with a collection of concrete Validator implementations. During the asset build process, the builder iterates through these validators, invoking each one to scrutinize a specific aspect of the NPC data. This approach decouples the validation logic from the asset construction logic, allowing for modular, extensible, and maintainable data integrity checks.

For example, concrete subclasses might include a ModelPathValidator, an AnimationNameValidator, or a FactionDataValidator. Each is responsible for a single, focused validation task.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of Validator are typically instantiated by a central factory or dependency injection container during server initialization. They are then registered with the primary NpcAssetBuilder service. They are not intended for direct instantiation by general game logic.
- **Scope:** Validator instances are designed to be long-lived and stateless. They persist for the entire server session and are reused across thousands of asset build operations.
- **Destruction:** These objects are destroyed only when the server shuts down and the application context is torn down.

## Internal State & Concurrency
- **State:** The Validator base class is stateless. Concrete implementations **must** be designed to be stateless. They should not contain any mutable fields that store information from previous validation calls. All necessary data for a validation operation must be passed in as method arguments.
- **Thread Safety:** This class is inherently thread-safe due to its lack of state. All subclasses **must** maintain this thread-safety guarantee. The asset build pipeline may be multi-threaded to improve performance, and stateful validators would introduce critical race conditions and data corruption.

**WARNING:** Introducing mutable state into a Validator subclass is a severe architectural violation and will lead to unpredictable behavior and non-deterministic asset build failures.

## API Surface
As an abstract class, Validator defines a contract for its children. The primary interaction point is a single validation method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate(context) | ValidationResult | O(N) | Executes a specific validation check against the data in the provided context. Returns a result object indicating success or failure with detailed error messages. |

*Note: The exact signature is inferred, but will involve passing an asset context or data object.*

## Integration Patterns

### Standard Usage
Developers do not interact with the Validator base class directly. The standard pattern is to create a new, concrete implementation for a specific piece of validation logic and register it with the engine.

```java
// 1. Implement a concrete validator
public class ModelExistsValidator extends Validator {
    @Override
    public ValidationResult validate(AssetBuildContext context) {
        String modelPath = context.getNpcData().getModelPath();
        if (!AssetSystem.exists(modelPath)) {
            return ValidationResult.failure("NPC model not found at path: " + modelPath);
        }
        return ValidationResult.success();
    }
}

// 2. Register the validator with the build system (conceptual)
// This typically happens once at server startup.
NpcAssetBuilder.registerValidator(new ModelExistsValidator());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new Validator()`. This is an abstract class and cannot be instantiated.
- **Stateful Implementations:** Do not store data from a validation call in a field. This breaks thread safety and reusability.
- **Complex Logic:** Avoid putting multiple, unrelated checks into a single Validator. Each class should adhere to the Single Responsibility Principle. For example, do not check for model existence and animation validity in the same validator.

## Data Pipeline
The Validator sits in the middle of the NPC asset compilation pipeline, acting as a gatekeeper for data quality.

> Flow:
> Raw NPC Definition (JSON/HOCON) -> AssetLoader -> NpcAssetBuildContext -> **(ModelExistsValidator, NameValidator, etc.)** -> ValidationResult -> NpcAssetCompiler -> Finalized Server NPC Asset

