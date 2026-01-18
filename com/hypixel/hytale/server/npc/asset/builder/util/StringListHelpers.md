---
description: Architectural reference for StringListHelpers
---

# StringListHelpers

**Package:** com.hypixel.hytale.server.npc.asset.builder.util
**Type:** Utility

## Definition
```java
// Signature
public class StringListHelpers {
```

## Architecture & Concepts
StringListHelpers is a stateless, static utility class that serves as a foundational component for data serialization and deserialization within the server's asset pipeline. Its primary architectural role is to provide a standardized, high-performance bridge between flat string representations and complex in-memory collection types like Lists, Sets, and nested Lists.

This class enforces a consistent data encoding scheme across different systems, particularly for NPC and asset configuration files. By centralizing this logic, the engine avoids divergent parsing implementations and ensures that data read from disk or network packets is reliably converted into structured data that the game logic can operate on.

The core conventions established by this class are:
*   **List Delimiter:** A comma, semicolon, or whitespace character (`[,; \t]`) separates items in a single list.
*   **Nested List Delimiter:** A pipe character (`|`) separates entire lists within a list of lists.

Its location within the `npc.asset.builder` package strongly indicates its primary use case is parsing complex properties from asset definitions during server bootstrap or asset hot-reloading.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its private constructor prevents the creation of instances, enforcing a purely static usage pattern. The class is loaded into the JVM by the ClassLoader when first referenced.
- **Scope:** The class and its static methods are available at the application level and persist for the entire lifetime of the server process.
- **Destruction:** The class is unloaded from the JVM when the application terminates. No manual resource management or cleanup is necessary.

## Internal State & Concurrency
- **State:** StringListHelpers is stateless from a consumer's perspective. Internally, it maintains two private static final Pattern objects (`listSplitter`, `listListSplitter`). These compiled regular expressions are initialized once at class-load time and are immutable. This is a critical performance optimization that prevents costly regex compilation on every method call.
- **Thread Safety:** This class is unconditionally thread-safe. All methods are pure functions that operate solely on their inputs without modifying any shared state. It can be safely invoked from multiple threads concurrently without requiring any external synchronization.

**WARNING:** While the methods themselves are thread-safe, methods that accept a `Collection` as a destination for results (e.g., `splitToStringList(String, Function, Collection)`) depend on the thread safety of the provided collection. Callers are responsible for ensuring that the target collection is thread-safe if it is accessed from multiple threads.

## API Surface
The public API provides a symmetric set of methods for converting from collections to strings (joining) and from strings to collections (splitting).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| stringListToString(Collection) | String | O(N) | Joins a collection of strings into a single, comma-delimited string. |
| splitToStringList(String, Function) | List | O(M) | Splits a delimited string into a List of strings. Null-safe. |
| stringListListToString(Collection) | String | O(T) | Joins a collection of string collections into a pipe-delimited string. |
| splitToStringListList(String, Function) | List | O(M) | Splits a string into a List of Lists, using pipes as the outer delimiter. |
| splitToStringSet(String, Function) | Set | O(M) | Splits a delimited string directly into a Set of unique strings. |

*N = number of elements in the input collection. M = length of the input string. T = total number of elements across all nested collections.*

## Integration Patterns

### Standard Usage
This class is intended for direct, static invocation when parsing data from configuration files, database records, or any other string-based source.

```java
// Example: Parsing NPC tags from an asset file property.
String rawFactionTags = "undead; skeleton| minion, archer";
List<List<String>> tagGroups = StringListHelpers.splitToStringListList(rawFactionTags, String::toLowerCase);

// tagGroups now contains:
// [ ["undead", "skeleton"], ["minion", "archer"] ]

String singleGroup = "player, hostile, melee";
Set<String> tags = StringListHelpers.splitToStringSet(singleGroup, null);

// tags now contains: ["player", "hostile", "melee"]
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Do not attempt to create an instance of StringListHelpers using `new` or through reflection. The class is designed to be purely static.
- **Complex Logic in Mappers:** The `mapper` and `transform` function arguments should be reserved for simple, fast transformations like `String::trim` or `String::toLowerCase`. Avoid embedding complex business logic, I/O operations, or network calls within these functions, as it violates the class's purpose and can introduce severe performance bottlenecks.
- **Parsing Unstructured Data:** This utility is designed for a specific, well-defined delimiter convention. Do not use it to parse natural language or other unstructured text, as the results will be unpredictable.

## Data Pipeline
StringListHelpers acts as a transformation stage within a larger data ingestion pipeline, typically during asset loading. It does not orchestrate the pipeline but is a critical step within it.

> Flow:
> Asset File on Disk (e.g., npc_skeleton.json) -> File I/O -> JSON Parser -> Raw String Property (e.g., "tags: 'undead, melee'") -> **StringListHelpers**.splitToStringSet -> In-Memory NPC Asset Object (e.g., `npc.getTags().add("undead")`)

