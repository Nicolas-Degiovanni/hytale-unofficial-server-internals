---
description: Architectural reference for FastRandom
---

# FastRandom

**Package:** com.hypixel.hytale.math.util
**Type:** Transient

## Definition
```java
// Signature
public class FastRandom extends Random {
```

## Architecture & Concepts
FastRandom is a high-performance, non-thread-safe pseudo-random number generator (PRNG). It serves as a specialized replacement for the standard Java `Random` and `ThreadLocalRandom` classes in performance-critical contexts where the overhead of synchronization is unacceptable.

The implementation is a classic Linear Congruential Generator (LCG), using the same constants for its multiplier, addend, and mask as `java.util.Random`. The primary architectural distinction is the deliberate omission of thread-safety mechanisms. While `java.util.Random` uses an `AtomicLong` to manage its seed, FastRandom uses a primitive `long`. This design choice trades safety for raw speed, making it suitable for single-threaded game logic such as particle simulation, procedural animation, or AI decision-making where a deterministic sequence is often desired and performance is paramount.

The class explicitly disables the `nextGaussian` method, signaling its role as a simplified, core PRNG focused exclusively on generating uniformly distributed integer bits.

### Lifecycle & Ownership
- **Creation:** Instantiated directly and on-demand by client code. It is not managed by a dependency injection framework or service locator. Typically, an object that requires its own deterministic random sequence (e.g., a WorldGenerator for a specific chunk, a particle emitter) will create and own a FastRandom instance.
- **Scope:** The lifetime of a FastRandom instance is bound to its owner. It is intended for local, contained use rather than as a global or session-scoped singleton.
- **Destruction:** The object is eligible for garbage collection as soon as its owner is no longer referenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The class is fundamentally mutable. Its entire state consists of a single `long` field: `seed`. Every call to the `next` method mutates this seed, which is essential for generating the subsequent number in the sequence.
- **Thread Safety:** **This class is NOT thread-safe.** The `seed` field is accessed and modified without any synchronization. If a single FastRandom instance is shared across multiple threads, severe data races will occur, corrupting the seed and destroying the statistical properties of the output. Each thread requiring a random number generator must have its own unique instance of FastRandom.

## API Surface
The public contract is inherited from `java.util.Random`, but with critical implementation differences.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| FastRandom() | constructor | O(1) | Creates a new generator, seeded with a value from ThreadLocalRandom. |
| FastRandom(long seed) | constructor | O(1) | Creates a new generator with a specific seed for deterministic sequences. |
| setSeed(long seed) | void | O(1) | Resets the generator's internal state to a specific seed. |
| next(int bits) | int | O(1) | The core generation method. Atomically advances the seed and returns the requested number of high-order bits. |
| nextGaussian() | double | O(1) | **Unsupported.** Throws UnsupportedOperationException. Do not call this method. |

## Integration Patterns

### Standard Usage
For deterministic, high-performance, and single-threaded random number generation.

```java
// Create a generator for a specific, repeatable task
long worldSeed = 12345L;
FastRandom worldGenRandom = new FastRandom(worldSeed);

// Use it to make decisions
for (int i = 0; i < 100; i++) {
    if (worldGenRandom.nextBoolean()) {
        // place tree
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instance Across Threads:** The most critical anti-pattern. Sharing one instance will lead to a corrupted state and unpredictable behavior.

    ```java
    // DO NOT DO THIS
    FastRandom shared = new FastRandom();
    
    // Thread 1
    new Thread(() -> System.out.println(shared.nextInt())).start();
    
    // Thread 2
    new Thread(() -> System.out.println(shared.nextInt())).start();
    ```

- **Security-Sensitive Operations:** The LCG algorithm is predictable. Do not use FastRandom for cryptography, generating session tokens, or any other security-related purpose. Use `java.security.SecureRandom` instead.

## Data Pipeline
FastRandom is a data source, not a processing component. Its internal data flow is a simple feedback loop.

> Flow:
> Internal `seed` -> **FastRandom.next()** -> (New `seed` stored internally, Random `int` returned)

