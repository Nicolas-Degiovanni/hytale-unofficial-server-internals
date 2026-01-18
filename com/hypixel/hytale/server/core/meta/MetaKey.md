---
description: Architectural reference for MetaKey
---

# MetaKey

**Package:** com.hypixel.hytale.server.core.meta
**Type:** Handle / Value Object

## Definition
```java
// Signature
public class MetaKey<T> {
```

## Architecture & Concepts
The MetaKey class is a foundational component of the server's metadata system. It functions as a type-safe, unique handle for attaching arbitrary data to game objects, such as entities, players, or worlds. It is not a container for data itself; rather, it is an immutable pointer to a specific metadata slot.

The core design principle is to provide a robust, compile-time safe mechanism for associating values with objects without modifying the object's class definition. The generic type parameter T ensures that any attempt to get or set metadata using this key is type-checked by the compiler, preventing a significant class of runtime errors.

Internally, each MetaKey is backed by a simple integer ID. This ID is used for highly efficient lookups within metadata containers, which are typically implemented as array-backed maps. The uniqueness of these IDs is guaranteed by a central, package-private factory or registry, which is the sole authority for creating MetaKey instances. This pattern prevents key collisions and abstracts the performance-critical integer ID behind a safe, developer-friendly API.

## Lifecycle & Ownership
-   **Creation:** MetaKey instances are created exclusively by a central registry system within the `com.hypixel.hytale.server.core.meta` package. The constructor is intentionally package-private to enforce this pattern. Developers do not instantiate MetaKey directly; they declare static final fields and initialize them by calling the registry.
-   **Scope:** Keys are designed to be static constants that persist for the entire lifetime of the server. They are defined once at server startup and reused throughout the codebase.
-   **Destruction:** As static final references, MetaKeys are not typically subject to garbage collection until the server shuts down and their defining class is unloaded.

## Internal State & Concurrency
-   **State:** Deeply immutable. The internal state consists of a single `private final int id`, which is assigned at creation and can never be changed.
-   **Thread Safety:** Inherently thread-safe. Due to its immutability, a MetaKey instance can be safely shared, published, and used across any number of threads without requiring locks or any other synchronization primitives.

## API Surface
The public contract is minimal, focusing on its function as an identifier within collections.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the unique, low-level integer ID for this key. Primarily for internal use by metadata containers. |
| equals(Object) | boolean | O(1) | Compares two keys for equality based solely on their internal integer ID. |
| hashCode() | int | O(1) | Returns the internal integer ID, ensuring keys work correctly in hash-based collections. |

## Integration Patterns

### Standard Usage
The correct pattern is to define keys as static final fields, obtaining them from the central registry. These keys are then used with metadata-aware objects to get and set typed data.

```java
// In a constants or component class
public static final MetaKey<Integer> PLAYER_HEALTH = MetaKeyRegistry.create("player_health");
public static final MetaKey<String> PLAYER_DISPLAY_NAME = MetaKeyRegistry.create("player_display_name");

// In game logic
public void updatePlayer(Player player) {
    player.setMeta(PLAYER_HEALTH, 100);
    String name = player.getMeta(PLAYER_DISPLAY_NAME).orElse("DefaultName");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The package-private constructor prevents `new MetaKey()`. Any attempt to bypass this using reflection will break the uniqueness guarantee of the metadata system and lead to unpredictable behavior.
-   **Dynamic Key Creation:** Do not create keys on-the-fly in game loops or frequently executed code. Keys are meant to be static, well-defined constants. Creating them dynamically constitutes a memory leak, as the backing registry will never release them.
-   **Reference Comparison:** While it may coincidentally work, do not compare keys using the `==` operator. Always use the `equals` method to guarantee correct behavior.

## Data Pipeline
MetaKey does not process data itself. It acts as a routing key to direct data into and out of a metadata storage system.

> Flow:
> `MetaKeyRegistry` creates and issues a unique **`MetaKey`** -> The key is used to write a value into an object's `MetadataContainer` -> The same **`MetaKey`** is later used to retrieve the type-safe value from the `MetadataContainer`.

