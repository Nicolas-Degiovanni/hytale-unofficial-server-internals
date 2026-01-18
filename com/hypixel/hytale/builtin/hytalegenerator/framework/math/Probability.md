---
description: Architectural reference for Probability
---

# Probability

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class Probability {
```

## Architecture & Concepts
The Probability class is a foundational, stateless utility within the Hytale world generation framework. Its primary architectural role is to provide **deterministic, reproducible random outcomes**. This is a cornerstone of procedural content generation, ensuring that a world generated with a specific seed will be identical every time it is created.

Unlike a standard random number generator that might use a system clock for its seed, every operation in this class is explicitly seeded. This design guarantees that probabilistic decisions—such as whether to place a tree, spawn a creature, or generate a cave system—are repeatable across all clients and server instances for a given world seed. It acts as a low-level mathematical primitive consumed by higher-level generator services.

### Lifecycle & Ownership
- **Creation:** As a static utility class, Probability is never instantiated. Its bytecode is loaded by the JVM ClassLoader on first access.
- **Scope:** The class is available for the entire application lifetime once loaded.
- **Destruction:** It is unloaded by the ClassLoader during JVM shutdown. There is no manual memory management or cleanup required.

## Internal State & Concurrency
- **State:** The Probability class is entirely stateless. It contains no member variables and all methods are static. Each invocation of the *of* method creates a new, short-lived `java.util.Random` object on the stack, ensuring that no state is shared between calls.
- **Thread Safety:** This class is inherently thread-safe. Because each call is a pure function of its inputs and creates its own isolated `Random` instance, it can be safely invoked from multiple world generation threads simultaneously without locks or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| of(double chance, long seed) | boolean | O(1) | Evaluates a probabilistic event deterministically. Returns true if a random double generated from the seed is less than the specified chance. |

## Integration Patterns

### Standard Usage
This utility should be invoked by world generation algorithms whenever a seeded, probabilistic decision is required. The seed should typically be a combination of the global world seed and context-specific data, such as chunk coordinates or a feature hash, to ensure unique but deterministic outcomes for each location.

```java
// Example: Deciding whether to place a rare ore vein in a chunk
long worldSeed = 12345L;
int chunkX = 10;
int chunkZ = -5;

// Combine world seed with chunk coordinates to create a deterministic local seed
long chunkSeed = worldSeed ^ (chunkX << 32) ^ chunkZ;

if (Probability.of(0.05, chunkSeed)) {
    // Generate the rare ore vein for this chunk
}
```

### Anti-Patterns (Do NOT do this)
- **Non-Deterministic Seeding:** Never use a volatile seed like `System.currentTimeMillis()` with this class. Doing so violates its core architectural purpose of providing reproducible results and will lead to divergent world generation.
- **Performance-Intensive Loops:** The `of` method instantiates a new `java.util.Random` object on every call. While acceptable for most use cases, invoking it millions of times in a tight, performance-critical loop can introduce unnecessary object allocation overhead. For such scenarios, consider creating a single `Random` instance and reusing it for the duration of the high-frequency task.

## Data Pipeline
The data flow for this component is simple and transactional. It does not participate in a larger asynchronous pipeline but rather serves as a synchronous function call within one.

> Flow:
> World Seed + Positional Data -> Combined Seed (long) -> **Probability.of()** -> Boolean Decision -> Generator Logic

