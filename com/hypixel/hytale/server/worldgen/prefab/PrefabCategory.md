---
description: Architectural reference for PrefabCategory
---

# PrefabCategory

**Package:** com.hypixel.hytale.server.worldgen.prefab
**Type:** Utility / Data Transfer Object (DTO)

## Definition
```java
// Signature
public record PrefabCategory(String name, int priority) {
```

## Architecture & Concepts
PrefabCategory is an immutable data record that defines a classification and placement priority for world generation prefabs. In the Hytale world generation pipeline, prefabs (pre-designed structures or features) must be ordered to resolve conflicts and determine placement sequence. This class provides that ordering mechanism via an integer priority.

It functions as a core component of a data-driven design. Categories are not hard-coded but are loaded from a central configuration file, `PrefabCategories.json`. This allows designers to define new prefab types and adjust their world generation behavior without modifying engine code.

The class includes two critical sentinel values:
*   **NONE:** Represents the absolute lowest priority. Prefabs in this category are likely to be placed first and be overwritten by almost any other feature.
*   **UNIQUE:** Represents the absolute highest priority. This is reserved for critical, one-of-a-kind prefabs that must not be overwritten, such as quest-specific landmarks.

The primary logic is encapsulated in the static `parse` method, which acts as a deserializer. It consumes a GSON JsonElement and uses a `BiConsumer` callback to populate a collection managed by a higher-level service, typically a central registry for prefab metadata.

## Lifecycle & Ownership
-   **Creation:** Instances are created under two conditions:
    1.  Statically at class-load time for the `NONE` and `UNIQUE` sentinel instances.
    2.  Dynamically during server initialization by a manager class (e.g., a `PrefabRegistry`) that invokes the static `parse` method. This method is the sole entry point for creating instances from configuration data.
-   **Scope:** The static instances are application-scoped and exist for the lifetime of the JVM. Dynamically created instances persist for the entire server session, held as values within a central registry map.
-   **Destruction:** Instances are subject to standard garbage collection. They are effectively destroyed when the server shuts down and the registry that holds them is cleared.

## Internal State & Concurrency
-   **State:** **Immutable**. As a Java record, its fields (`name`, `priority`) are final and cannot be modified after construction. An instance of PrefabCategory represents a constant value.
-   **Thread Safety:** **Fully thread-safe**. Its immutability guarantees that it can be safely read and shared across any number of threads without synchronization. The static `parse` method is also thread-safe as it is a pure function with no side effects on shared static state.

## API Surface
The public API is minimal, reflecting its role as a simple data holder.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| name() | String | O(1) | Accessor for the category's unique identifier. |
| priority() | int | O(1) | Accessor for the category's placement priority. |
| parse(json, consumer) | static void | O(N) | Deserializes a JSON object into PrefabCategory instances. N is the number of categories in the JSON. Throws an Error on malformed data. |

## Integration Patterns

### Standard Usage
The `parse` method is designed to be called by a service responsible for loading game assets. The service reads the JSON file, parses it, and then passes the resulting `JsonElement` and a consumer to populate its internal cache.

```java
// Example from a hypothetical PrefabRegistry service
private final Map<String, PrefabCategory> categories = new ConcurrentHashMap<>();

public void loadCategories(JsonElement categoryJson) {
    // The consumer is a method reference that adds the parsed
    // category to the registry's internal map.
    PrefabCategory.parse(categoryJson, this.categories::put);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually create instances with `new PrefabCategory()`. The system is designed to be entirely data-driven from `PrefabCategories.json`. Ad-hoc instances will not be known to the world generator and will cause inconsistent or undefined behavior.
-   **Incorrect Priority:** Assigning priorities outside the logical range or misusing the `MIN_PRIORITY` and `MAX_PRIORITY` constants can severely disrupt the world generation process, leading to incorrect feature placement or overlap.

## Data Pipeline
This class is an early stage in the prefab data ingestion pipeline. It translates raw configuration data into a strongly-typed, in-memory representation for the rest of the engine to use.

> Flow:
> `PrefabCategories.json` (File on Disk) -> Asset Loading Service -> GSON Parser -> `JsonElement` -> **PrefabCategory.parse** -> `PrefabCategory` instances (In-Memory Registry) -> World Generator Logic

