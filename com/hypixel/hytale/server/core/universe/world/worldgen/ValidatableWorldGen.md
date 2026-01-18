---
description: Architectural reference for ValidatableWorldGen
---

# ValidatableWorldGen

**Package:** com.hypixel.hytale.server.core.universe.world.worldgen
**Type:** Interface

## Definition
```java
// Signature
public interface ValidatableWorldGen {
```

## Architecture & Concepts
The ValidatableWorldGen interface defines a formal contract for world generation components that require a pre-execution validation step. It serves as a critical checkpoint within the world generation pipeline, allowing the system to verify the integrity, configuration, and environmental compatibility of a generator before committing significant computational resources to its execution.

Implementations of this interface are typically complex world generators, procedural rule sets, or data-driven biome assemblers. The core architectural purpose is to decouple the validation logic from the generation logic, enabling the world engine to query the state of a generator without invoking its full, and often expensive, generation process. This pattern is essential for preventing catastrophic failures, corrupted world data, and wasted server cycles due to misconfiguration.

## Lifecycle of Implementations
- **Creation:** Concrete implementations are typically instantiated by a WorldGeneratorFactory or loaded via a configuration service when a world is being initialized or a new dimension is being prepared.
- **Scope:** The lifecycle of an implementing object is tied to the world generation session for a specific region or dimension. It is generally short-lived, existing only for the duration of the setup and validation phase.
- **Destruction:** The object is eligible for garbage collection once the validation check is complete and the primary world generator has been fully configured and handed off to the execution engine.

## Internal State & Concurrency
- **State:** Implementations are expected to be stateful, holding the configuration and parameters that need to be validated. The state should be considered immutable *during* the validation call itself.
- **Thread Safety:** The validate method must be implemented as a thread-safe, read-only operation. It is frequently called from the main world generation orchestrator thread but must not assume this exclusivity. Any internal state it reads must be protected against concurrent modification if the implementing class is accessible from other threads.

**WARNING:** Implementations of validate must be non-blocking and computationally inexpensive. Performing file I/O, network requests, or heavy computations within this method will severely degrade world loading performance and is considered a critical anti-pattern.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validate() | boolean | O(N) | Executes a series of checks against the generator's internal state and configuration. Returns true if the generator is ready for execution, false otherwise. |

## Integration Patterns

### Standard Usage
This interface is not used directly by typical game logic. It is consumed by the core world generation service to gate the execution of a generator.

```java
// System-level code within the world generation engine
WorldGenerator potentialGenerator = generatorFactory.createFromConfig(config);

if (potentialGenerator instanceof ValidatableWorldGen) {
    if (!((ValidatableWorldGen) potentialGenerator).validate()) {
        // Halt world creation and log a critical error
        throw new WorldGenValidationException("Generator failed validation: " + config.getName());
    }
}

// Proceed with expensive world generation
worldEngine.run(potentialGenerator);
```

### Anti-Patterns (Do NOT do this)
- **Expensive Validation:** Do not implement the validate method to perform the actual generation. It is a pre-check, not the main operation.
- **Ignoring The Result:** Systems that retrieve a ValidatableWorldGen must never ignore a false return value. Proceeding with generation after a failed validation leads to undefined behavior and probable server instability.
- **State Modification:** The validate method must not modify the internal state of the implementing object. It is a read-only contract.

## Data Pipeline

The interface acts as a control-flow gate rather than a data processing component.

> Flow:
> World Configuration -> Generator Instantiation -> **Validation Check (ValidatableWorldGen.validate)** -> [STOP on false] -> World Generation Execution

