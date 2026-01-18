---
description: Architectural reference for CalculationResult
---

# CalculationResult

**Package:** com.hypixel.hytale.server.core.universe.world.lighting
**Type:** Enum

## Definition
```java
// Signature
public enum CalculationResult {
```

## Architecture & Concepts
CalculationResult is an enumeration that defines the finite state machine for a single lighting calculation task, typically operating on a world chunk. It is not a service or manager, but rather a set of constant values that represent the outcome or current status of a complex, asynchronous operation within the server's lighting engine.

This enum is critical for managing the dependency graph of lighting calculations. Because light propagates between chunks, a chunk's lighting cannot be finalized until its neighbors are in a compatible state. CalculationResult provides the vocabulary for the lighting scheduler to orchestrate this multi-step, interdependent process, preventing race conditions and ensuring correctness.

## Lifecycle & Ownership
- **Creation:** Instances of this enum are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. They are initialized once when the `CalculationResult` class is first referenced.
- **Scope:** These instances are static and exist for the entire lifetime of the application. They are effectively global, immutable singletons.
- **Destruction:** The enum instances are garbage collected only when the defining class loader is unloaded, which typically occurs at server shutdown.

## Internal State & Concurrency
- **State:** Inherently immutable. The state of an enum constant can never be changed after its creation by the JVM.
- **Thread Safety:** This enum is unconditionally thread-safe. Its instances can be safely passed between and read from any thread without synchronization. This is a fundamental guarantee of the Java language specification.

## API Surface
The API consists of the enum constants themselves, which are used for state representation and comparison.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NOT_LOADED | CalculationResult | Constant | Represents a chunk whose data is not yet available for lighting calculation. The task cannot begin. |
| DONE | CalculationResult | Constant | The terminal success state. Lighting for the target chunk is fully calculated and considered stable. |
| INVALIDATED | CalculationResult | Constant | Indicates the calculation was completed but has since been invalidated by an external change (e.g., a block modification). The task must be re-queued. |
| WAITING_FOR_NEIGHBOUR | CalculationResult | Constant | Indicates the calculation is paused because it depends on a neighboring chunk that is not yet in the DONE state. The scheduler will re-attempt this task later. |

## Integration Patterns

### Standard Usage
This enum is primarily used as a return type for lighting calculation methods to signal the outcome to a scheduler or worker system. The caller is expected to use a switch statement or conditional logic to handle the result.

```java
// A scheduler processing lighting tasks for a world chunk
LightTask task = getNextTask();
CalculationResult result = lightCalculator.processChunk(task.getChunk());

switch (result) {
    case DONE:
        task.markComplete();
        worldMesher.queueForRemesh(task.getChunk());
        break;
    case WAITING_FOR_NEIGHBOUR:
        // Re-queue the task to be attempted later
        scheduler.requeueWithDelay(task);
        break;
    case INVALIDATED:
        // Re-queue immediately with high priority
        scheduler.requeueImmediately(task);
        break;
    case NOT_LOADED:
        // The chunk was likely unloaded, discard the task
        task.discard();
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on `result.ordinal()` for logic. The order of enum declarations can change, which would break any logic based on their integer value. Always compare instances directly.
- **Null Usage:** A method that returns a CalculationResult should never return null. The `NOT_LOADED` state explicitly handles cases where a calculation cannot proceed. Using null introduces ambiguity and risks NullPointerExceptions.
- **String Comparison:** Never use `result.toString().equals("DONE")`. This is inefficient, error-prone, and defeats the purpose of a type-safe enum.

## Data Pipeline
CalculationResult acts as a control signal, not a data container. It directs the flow of a chunk through the server's world-generation and update pipeline.

> Flow:
> Block Update -> Lighting Task Queued -> Light Calculator -> **CalculationResult**
>
> - If **DONE**: Chunk -> World Mesher -> Network Packet
> - If **WAITING_FOR_NEIGHBOUR**: Chunk -> Re-queued in Light Scheduler
> - If **INVALIDATED**: Chunk -> Re-queued in Light Scheduler

