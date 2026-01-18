---
description: Architectural reference for Scanner
---

# Scanner

**Package:** com.hypixel.hytale.builtin.hytalegenerator.scanners
**Type:** Abstract Base Class / Strategy

## Definition
```java
// Signature
public abstract class Scanner {
```

## Architecture & Concepts
The Scanner is an abstract base class that defines a core component of the procedural world generation pipeline. It embodies the **Strategy** design pattern, encapsulating algorithms for identifying valid placement locations for a given Pattern within a VoxelSpace.

In essence, a Scanner answers the question: "Given this region of the world and this object I want to place, where are all the valid coordinates to do so?"

It serves as the fundamental contract for various scanning implementations, such as finding surfaces, locating specific material types, or identifying volumes of empty space. The world generation orchestrator selects and uses a concrete Scanner implementation based on the requirements of the Pattern being placed. This decouples the high-level generation logic from the low-level details of position validation.

The presence of a WorkerIndexer Id within its execution Context is a critical design element, indicating that Scanners are designed to be executed in a highly parallelized, multi-threaded environment.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of Scanner are instantiated by higher-level world generation components, such as a Placer or a ZoneGenerator. The static factory method noScanner provides a null-object implementation for cases where no scanning logic is required.
- **Scope:** A Scanner instance is typically transient and short-lived. It is created for a specific generation task, used to scan one or more regions, and then becomes eligible for garbage collection. It does not persist between generation phases.
- **Destruction:** Managed by the Java Garbage Collector. There are no explicit cleanup or teardown methods, as Scanners are expected to be stateless or manage only transient state related to a single scan operation.

## Internal State & Concurrency
- **State:** The abstract Scanner class is stateless. Concrete implementations should be designed to be either stateless or to encapsulate their state in a re-entrant manner. All necessary contextual state for an operation is passed via the Context parameter in the scan method.
- **Thread Safety:** Implementations of Scanner **must be thread-safe**. The world generation system executes Scanners across multiple worker threads simultaneously. The inclusion of WorkerIndexer.Id in the Context object implies that operations may need to be deterministic based on the worker, or that thread-local resources might be used. Any shared mutable state within a Scanner implementation would require explicit synchronization, which is heavily discouraged. The preferred pattern is to rely solely on the provided Context for input.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| scan(Context) | List<Vector3i> | Varies | Abstract method. Executes the core scanning algorithm and returns a list of valid 3D integer coordinates. |
| scanSpace() | SpaceSize | O(1) | Abstract method. Returns the bounding box or volume that this scanner considers for its operations. |
| readSpaceWith(Pattern) | SpaceSize | O(1) | Calculates the combined bounding volume of the scanner's space and a given pattern's space. |
| noScanner() | Scanner | O(1) | Static factory method. Returns a singleton, no-op implementation of Scanner that always returns an empty list. |

## Integration Patterns

### Standard Usage
A Scanner is used by a generation orchestrator to find placement locations. The orchestrator is responsible for constructing the Context object, which represents the current state of the world chunk being processed.

```java
// Assume 'generator' is a higher-level world generation service
// and 'surfaceScanner' is a concrete implementation of Scanner.

VoxelSpace<Material> worldChunk = generator.getVoxelSpaceForRegion(region);
Pattern treePattern = assetManager.getPattern("oak_tree");
WorkerIndexer.Id workerId = threadPool.getCurrentWorkerId();
Vector3i scanOrigin = new Vector3i(0, 64, 0);

Scanner.Context scanContext = new Scanner.Context(scanOrigin, treePattern, worldChunk, workerId);
List<Vector3i> validPositions = surfaceScanner.scan(scanContext);

for (Vector3i pos : validPositions) {
    generator.applyPattern(worldChunk, treePattern, pos);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Avoid creating Scanner subclasses that store mutable state across multiple calls to scan. This is not thread-safe and will lead to unpredictable generation results.
- **Ignoring Context:** Do not disregard the parameters within the Context object. The position, pattern, and workerId are all essential for correct, deterministic, and performant operation.
- **Long-Lived Instances:** Do not cache and reuse Scanner instances across major generation phases if their configuration is meant to be dynamic. They are designed to be lightweight and created as needed.

## Data Pipeline
The Scanner acts as a filter and locator within the broader world generation data flow.

> Flow:
> Generation Orchestrator -> **Scanner.scan(Context)** -> List of Vector3i -> Pattern Applicator -> Modified VoxelSpace

