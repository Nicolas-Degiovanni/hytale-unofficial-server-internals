---
description: Architectural reference for DefaultMap
---

# DefaultMap

**Package:** com.hypixel.hytale.common.map
**Type:** Utility

## Definition
```java
// Signature
public class DefaultMap<K, V> implements Map<K, V> {
```

## Architecture & Concepts
DefaultMap is a specialized data structure that implements the Decorator Pattern. It wraps a standard Java Map implementation (the *delegate*), augmenting its behavior to provide a default value when a requested key is not present. This eliminates the need for repetitive null-checking and branching logic in consuming code, leading to cleaner and more predictable data access patterns.

This component is fundamental for systems where the absence of a value should be treated as a known, non-exceptional state. For example, it is commonly used for entity property maps, configuration stores, or resource caches where a fallback or default asset is preferable to a null reference.

The class also provides configurable policies for key replacement and null key handling, allowing it to serve as a flexible, write-once registry or a conventional mutable map.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor: `new DefaultMap(...)`. It is a transient object, not managed by a central service locator or dependency injection framework.
- **Scope:** The lifecycle of a DefaultMap instance is bound to the object that creates and holds a reference to it. It exists for as long as its owner does.
- **Destruction:** The object is eligible for garbage collection when all references to it are released. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The DefaultMap is a mutable state container. Its internal state is composed of the wrapped delegate map, the configured default value, and its behavior flags. The default value itself can be modified after instantiation via `setDefaultValue`.

- **Thread Safety:** **This class is not thread-safe.** All operations are delegated to the underlying map. If the delegate (such as the default HashMap) is not thread-safe, then DefaultMap is also not thread-safe. All concurrent access must be managed by external synchronization mechanisms. Failure to do so will result in non-deterministic behavior, including potential `ConcurrentModificationException` and data corruption.

## API Surface
The API fully implements the standard Java Map interface, with behavior modified by its configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DefaultMap(defaultValue, delegate, ...) | constructor | O(1) | Creates a new instance wrapping a delegate map. |
| get(key) | V | O(1) | Retrieves the value for a key. If the key is absent or its value is null, returns the configured default value. |
| put(key, value) | V | O(1) | Associates a value with a key. If `allowReplacing` is false, this method will throw an `IllegalArgumentException` if the key already exists. |
| setDefaultValue(defaultValue) | void | O(1) | Modifies the default value returned for missing keys. This affects all subsequent `get` calls. |
| getDefaultValue() | V | O(1) | Returns the currently configured default value. |
| getDelegate() | Map<K, V> | O(1) | Provides direct, unwrapped access to the underlying delegate map. |

## Integration Patterns

### Standard Usage
The primary use case is to simplify code that handles optional or default-able values.

```java
// Create a map where any missing key defaults to the integer 0.
// The delegate is a standard HashMap.
Map<String, Integer> playerScores = new DefaultMap<>(0);

playerScores.put("PlayerA", 150);

// Accessing an existing key works as expected.
int scoreA = playerScores.get("PlayerA"); // Returns 150

// Accessing a non-existent key returns the default value, not null.
int scoreB = playerScores.get("PlayerB"); // Returns 0
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Accessing a DefaultMap from multiple threads without external locking is unsafe and will lead to data corruption, especially if the delegate is a HashMap.

    ```java
    // DANGEROUS: Race condition between threads
    DefaultMap<String, String> sharedConfig = new DefaultMap<>("default");
    new Thread(() -> sharedConfig.put("key", "value1")).start();
    new Thread(() -> sharedConfig.put("key", "value2")).start();
    ```

- **Misunderstanding Immutability:** Setting `allowReplacing` to false does not make the map's *values* immutable, it only prevents re-assigning a key. The objects stored as values can still be mutated if they are not themselves immutable.

- **Direct Delegate Manipulation:** Modifying the delegate map directly via `getDelegate()` bypasses the logic of the DefaultMap wrapper. This can lead to inconsistent state if not done carefully.

    ```java
    // WARNING: Bypasses the 'allowReplacing' check
    DefaultMap<String, Integer> map = new DefaultMap<>(0, new HashMap<>(), false);
    map.put("level", 10);
    
    // This will throw an exception because allowReplacing is false.
    // map.put("level", 20); 
    
    // This bypasses the check and modifies the map directly.
    map.getDelegate().put("level", 20); // State is now potentially inconsistent
    ```

## Data Pipeline
As a data structure, DefaultMap does not process a pipeline of data. Instead, it acts as a conditional branching point within a data retrieval flow.

> **Get Operation Flow:**
> Caller requests `get(key)` -> **DefaultMap** checks internal delegate for `key` ->
> 1. **(Key Found)** -> Delegate returns value -> **DefaultMap** returns value to Caller.
> 2. **(Key Not Found)** -> Delegate returns null -> **DefaultMap** returns its `defaultValue` to Caller.

