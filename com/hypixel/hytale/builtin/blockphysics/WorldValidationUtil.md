---
description: Architectural reference for WorldValidationUtil
---

# WorldValidationUtil

**Package:** com.hypixel.hytale.builtin.blockphysics
**Type:** Utility

## Definition
```java
// Signature
public class WorldValidationUtil {
```

## Architecture & Concepts
WorldValidationUtil is a stateless factory class responsible for creating specialized block validation logic. It does not perform validation itself; instead, its methods generate and return a `IPrefabBuffer.RawBlockConsumer` instance, which is a functional interface (lambda) containing the validation rules.

This design pattern decouples the system that iterates over blocks (such as a prefab processor or a world generator) from the specific validation checks being performed. The calling system is responsible for providing a stream of raw block data, and it executes the consumer provided by this utility for each block in the stream.

The primary consumer of this utility is any system that needs to verify the integrity of a large set of blocks before placing them into the world. By taking a `StringBuilder` as an output parameter, it avoids performance penalties associated with string allocation and concatenation inside a potentially hot loop that may process millions of blocks.

## Lifecycle & Ownership
As a static utility class, WorldValidationUtil is never instantiated and has no object lifecycle.

- **Creation:** The class is not instantiated. Its static methods are invoked directly. The returned `RawBlockConsumer` lambda is created on each call to a `blockValidator` method.
- **Scope:** The methods are globally accessible. The returned consumer is a lightweight object that captures its creation context (the `StringBuilder` and `ValidationOption` set). Its lifetime is typically scoped to a single, complete validation operation.
- **Destruction:** The generated `RawBlockConsumer` lambda is eligible for garbage collection as soon as the calling system finishes its block processing loop and discards the reference to it.

## Internal State & Concurrency
- **State:** This class is completely stateless. It holds no mutable or immutable fields.
- **Thread Safety:** The static factory methods are inherently thread-safe. However, the `RawBlockConsumer` instance they produce is **not thread-safe**. It mutates the captured `StringBuilder` instance. Concurrent execution of the same consumer instance by multiple threads will lead to race conditions and a corrupted `StringBuilder` output. Each validation thread must create its own consumer or use external synchronization, which is strongly discouraged.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| blockValidator(sb, options) | IPrefabBuffer.RawBlockConsumer | O(1) | Creates a block validation consumer with a world origin offset of (0,0,0). |
| blockValidator(offsetX, offsetY, offsetZ, sb, options) | IPrefabBuffer.RawBlockConsumer | O(1) | Creates a block validation consumer with a custom world origin offset for error reporting. |

## Integration Patterns

### Standard Usage
The intended pattern is to create the validator just before a block processing operation and pass it to the processor. The `StringBuilder` is then inspected after the operation is complete.

```java
// How a developer should normally use this
StringBuilder errorLog = new StringBuilder();
Set<ValidationOption> checks = Set.of(ValidationOption.BLOCKS, ValidationOption.BLOCK_STATES);

// 1. Create the validation logic
IPrefabBuffer.RawBlockConsumer<Void> validator = WorldValidationUtil.blockValidator(errorLog, checks);

// 2. Pass the validator to a system that iterates over blocks
// (Hypothetical example)
IPrefabBuffer myPrefab = getPrefabToPlace();
myPrefab.processBlocks(validator);

// 3. Check the results
if (errorLog.length() > 0) {
    System.err.println("Prefab validation failed:");
    System.err.print(errorLog.toString());
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Consumer:** Do not create a single `RawBlockConsumer` and share it across multiple threads that are processing blocks concurrently. This will cause severe race conditions on the underlying `StringBuilder`.
- **Reusing StringBuilder without Clearing:** If the same `StringBuilder` instance is used for multiple, sequential validation operations, it must be cleared (e.g., via `sb.setLength(0)`) between uses. Failure to do so will result in an accumulation of errors from all previous operations.

## Data Pipeline
This utility functions as a processing node within a larger data flow. It does not initiate data flow but rather provides the logic for a single transformation and validation step.

> Flow:
> Prefab Buffer or Chunk Data Stream -> **RawBlockConsumer (from WorldValidationUtil)** -> Appends to StringBuilder -> Final Error Report

