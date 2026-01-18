---
description: Architectural reference for CaseInsensitiveHashStrategy
---

# CaseInsensitiveHashStrategy

**Package:** com.hypixel.hytale.assetstore.map
**Type:** Singleton

## Definition
```java
// Signature
public class CaseInsensitiveHashStrategy<K> implements Strategy<K> {
```

## Architecture & Concepts
The CaseInsensitiveHashStrategy is a low-level, performance-oriented utility that provides a case-insensitive hashing and equality-checking policy for map-like data structures. It implements the `Strategy` interface from the fastutil library, allowing it to be injected directly into fastutil collections to override their default key-handling behavior.

Its primary role within the engine is to support asset management and resource lookup systems where identifiers, such as file paths or resource keys, must be treated as equivalent regardless of capitalization. For example, this allows the system to treat "Meshes/Player.hobj" and "meshes/player.hobj" as references to the same asset.

By implementing a custom `hashCode` function that normalizes characters to lowercase during hash calculation, it avoids the overhead of creating a new, lowercased String object for every lookup, which is critical in performance-sensitive loops like rendering or asset streaming.

## Lifecycle & Ownership
- **Creation:** The single instance of this class is created statically when the class is loaded by the Java Virtual Machine. The `getInstance` method provides global access to this pre-existing instance.
- **Scope:** As a static singleton, its lifecycle is tied to the application itself. It persists from class-loading time until the application terminates.
- **Destruction:** The object is garbage collected by the JVM upon application shutdown. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This class is completely stateless. It holds no instance-level or static data beyond its own singleton instance. Its behavior is determined entirely by the inputs to its methods.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. The `hashCode` and `equals` methods are pure functions and can be safely invoked by any number of concurrent threads without locks or synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | CaseInsensitiveHashStrategy | O(1) | Returns the globally shared singleton instance. |
| hashCode(K key) | int | O(N) | Calculates a hash code for the key. If the key is a String, the hash is computed case-insensitively. N is the length of the string. |
| equals(K a, K b) | boolean | O(N) | Compares two keys for equality. If both keys are Strings, the comparison is case-insensitive. N is the length of the shorter string. |

## Integration Patterns

### Standard Usage
This strategy is not meant to be used directly. Instead, it should be passed to the constructor of a fastutil map or set to define its keying behavior for the lifetime of the collection.

```java
// How a developer should normally use this
import it.unimi.dsi.fastutil.objects.Object2ObjectOpenHashMap;
import java.util.Map;

// Create a map that uses case-insensitive strings as keys
Strategy<String> strategy = CaseInsensitiveHashStrategy.getInstance();
Map<String, Asset> assetMap = new Object2ObjectOpenHashMap<>(strategy);

assetMap.put("Textures/Stone.png", stoneAsset);

// This lookup will succeed, returning the stoneAsset
Asset foundAsset = assetMap.get("textures/stone.png");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new CaseInsensitiveHashStrategy()`. This bypasses the singleton pattern, creating unnecessary objects and defeating the purpose of a shared, stateless utility. Always use the static `getInstance()` method.
- **Misapplied Context:** Avoid using this strategy for maps where keys are not Strings or where case-sensitivity is required. While the implementation falls back to default behavior for non-String types, its use signals a specific intent for case-insensitivity that would be misleading in other contexts.

## Data Pipeline
This component does not process a pipeline of data itself. Instead, it acts as a behavioral controller within the data lookup process of a collection.

> Flow:
> Map Lookup (`map.get(key)`) -> **CaseInsensitiveHashStrategy.hashCode(key)** -> Hash Bucket Identification -> **CaseInsensitiveHashStrategy.equals(key, candidateKey)** -> Return Value

