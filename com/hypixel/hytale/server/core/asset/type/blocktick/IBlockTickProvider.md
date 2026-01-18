---
description: Architectural reference for IBlockTickProvider
---

# IBlockTickProvider

**Package:** com.hypixel.hytale.server.core.asset.type.blocktick
**Type:** Interface / Strategy

## Definition
```java
// Signature
@FunctionalInterface
public interface IBlockTickProvider {
```

## Architecture & Concepts
The IBlockTickProvider interface defines a fundamental strategy contract within the server's block update system. Its sole responsibility is to act as a lookup mechanism, mapping a numerical block identifier to its corresponding update logic, encapsulated by a TickProcedure.

Architecturally, it serves as a critical decoupling layer. The core BlockTickScheduler, which manages *when* to tick blocks, is completely separated from the game-specific logic of *how* a block behaves when ticked. The scheduler queries a provider to retrieve the correct behavior, which it then executes.

The designation as a FunctionalInterface is a deliberate design choice, encouraging implementations to be simple, stateless lambda expressions or method references, particularly for dynamic or procedurally defined block behaviors.

The static `NONE` instance implements the Null Object Pattern. It provides a safe, non-null default for all blocks that do not have ticking logic, eliminating the need for consumers like the BlockTickScheduler to perform null checks on the provider itself.

## Lifecycle & Ownership
- **Creation:** Implementations are instantiated and aggregated during the server's asset loading phase. A central AssetManager or a similar registry will typically construct a primary provider that delegates to multiple sources (e.g., one for vanilla blocks, one for each loaded mod).
- **Scope:** The lifecycle of a provider is tightly bound to the assets it represents. A provider for vanilla game assets persists for the entire server session. Providers associated with mods or dynamic content are loaded and unloaded with their parent resources.
- **Destruction:** An instance is eligible for garbage collection when the world or asset context that holds a reference to it is unloaded or shut down.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Concrete implementations are expected to be effectively immutable after their initial construction. They typically cache a mapping of block IDs to TickProcedures (e.g., in an array or a `Map`) which is populated once and subsequently only read from.
- **Thread Safety:** **CRITICAL:** All implementations of this interface **must be thread-safe**. The server's block ticking engine may operate across multiple worker threads, querying the provider concurrently for different blocks in different world regions. Implementations must use concurrent collections or ensure their internal data structures are safely published and read-only after initialization to prevent race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTickProcedure(int blockId) | TickProcedure | O(1) | Retrieves the executable tick logic for a given block ID. Returns null if no procedure is defined for the ID. Implementations must be highly performant. |

## Integration Patterns

### Standard Usage
This interface is a low-level component consumed by the core server engine, not typically by gameplay logic developers. The BlockTickScheduler is its primary consumer.

```java
// Conceptual usage within a hypothetical BlockTickScheduler
// Note: This is a simplified example for illustration.

IBlockTickProvider blockTickProvider = worldAssets.getBlockTickProvider();

// For a block that needs ticking...
int blockIdToTick = 123;
TickProcedure procedure = blockTickProvider.getTickProcedure(blockIdToTick);

if (procedure != null) {
    // The scheduler now has the logic to execute.
    executionEngine.submit(procedure);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** A provider that modifies its internal mappings after the server has started is strictly forbidden. This introduces non-deterministic behavior and will cause severe, difficult-to-diagnose concurrency bugs.
- **Slow Lookups:** The getTickProcedure method is in the hot path of the server tick loop. Any implementation that performs disk I/O, network requests, or heavy computation will degrade server performance catastrophically. All lookups must be near-instantaneous memory lookups.
- **Ignoring the NONE constant:** For systems that have no ticking blocks, do not implement a custom provider that always returns null. Use the provided `IBlockTickProvider.NONE` singleton for improved readability, intent, and performance.

## Data Pipeline
This component acts as a lookup service rather than a data processor. The flow represents a query for executable logic.

> Flow:
> BlockTickScheduler -> **IBlockTickProvider.getTickProcedure(blockId)** -> TickProcedure -> Execution Engine

