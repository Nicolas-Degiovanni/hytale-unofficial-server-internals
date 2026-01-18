---
description: Architectural reference for RandomUtil
---

# RandomUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class RandomUtil {
```

## Architecture & Concepts
The RandomUtil class is a static, engine-wide utility for all operations involving randomness. It serves as a centralized and standardized provider for random number generation, weighted selections, and collection sampling. Its design makes a critical distinction between two primary use cases:

1.  **Performance-Sensitive Randomness:** For high-frequency, non-security-critical game logic such as particle effects, AI decisions, or procedural decoration, the class provides methods that internally leverage ThreadLocalRandom. This avoids lock contention and provides the highest possible performance for game loop operations.

2.  **Cryptographically-Secure Randomness:** For security-sensitive operations like generating unique identifiers, session keys, or anything requiring unpredictability, the class provides access to a thread-local instance of SecureRandom.

By centralizing these functions, RandomUtil ensures that developers use the correct type of random number generator for the task at hand and promotes consistent, thread-safe patterns across the entire codebase.

## Lifecycle & Ownership
- **Creation:** As a static utility class, RandomUtil is never instantiated. Its static members are initialized by the JVM ClassLoader when the class is first referenced. The internal SecureRandom instance is managed by a ThreadLocal, meaning each thread creates and holds its own instance upon first access via getSecureRandom.

- **Scope:** The class and its static methods are available for the entire application lifetime. The thread-local SecureRandom instances persist for the lifetime of their owning thread.

- **Destruction:** The class is unloaded when the application terminates. Individual SecureRandom instances are eligible for garbage collection when their parent thread is destroyed.

## Internal State & Concurrency
- **State:** The RandomUtil class itself is stateless. However, it manages a static ThreadLocal field, SECURE_RANDOM, which holds a stateful SecureRandom object for each thread. This design isolates state on a per-thread basis.

- **Thread Safety:** This class is inherently thread-safe. All methods are static and operate on provided parameters or on thread-local random number generators (ThreadLocalRandom or the ThreadLocal SecureRandom). This design completely avoids shared mutable state and eliminates the risk of race conditions or lock contention when used correctly.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| roll(roll, data, chances) | T | O(N) | Performs a weighted selection from an array. Throws AssertionError if the roll falls outside the cumulative chance range. |
| getSecureRandom() | SecureRandom | O(1) | Retrieves the cryptographically strong random number generator for the current thread. |
| selectRandom(arr, random) | T | O(1) | Selects a random element from the given array using the supplied Random instance. |
| selectRandom(list) | T | O(1) | Selects a random element from a List using the high-performance ThreadLocalRandom. Assumes O(1) list access. |

## Integration Patterns

### Standard Usage
For common, non-security-critical tasks like selecting a random item from a pre-defined list, use the simplest method available. This ensures optimal performance by using the internal ThreadLocalRandom.

```java
// Select a random monster type from a list for spawning
List<Monster> monsterTypes = ...;
Monster monsterToSpawn = RandomUtil.selectRandom(monsterTypes);
```

For weighted loot table calculations, use the roll method.

```java
// Determine loot drop from a weighted table
LootItem[] items = { LOOT_A, LOOT_B, LOOT_C };
int[] chances = { 70, 25, 5 }; // Corresponds to items array
int dieRoll = ThreadLocalRandom.current().nextInt(100);
LootItem result = RandomUtil.roll(dieRoll, items, chances);
```

### Anti-Patterns (Do NOT do this)
- **Misusing SecureRandom:** Do not call getSecureRandom for high-frequency game logic like AI pathing or animation selection. SecureRandom is significantly slower and is intended only for security-critical contexts.

- **Passing Shared Random Instances:** Avoid creating a single java.util.Random instance and passing it to RandomUtil methods from multiple threads. This will cause heavy lock contention and degrade performance. Let the utility manage thread-local instances or pass ThreadLocalRandom.current().

- **Incorrect Roll Values:** The roll and rollInt methods expect a zero-based integer roll. Passing a one-based value without adjustment will lead to off-by-one errors in selection logic.

