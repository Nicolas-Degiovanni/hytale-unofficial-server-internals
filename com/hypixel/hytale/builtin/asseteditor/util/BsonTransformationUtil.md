---
description: Architectural reference for BsonTransformationUtil
---

# BsonTransformationUtil

**Package:** com.hypixel.hytale.builtin.asseteditor.util
**Type:** Utility

## Definition
```java
// Signature
public class BsonTransformationUtil {
```

## Architecture & Concepts

The BsonTransformationUtil is a stateless, static utility class designed for the deep, targeted manipulation of BsonDocument objects. It serves as a high-level abstraction over the complex and error-prone process of traversing and modifying nested BSON structures, which can contain a mix of documents (key-value maps) and arrays.

The core architectural concept is path-based addressing. A path, represented as an array of strings, uniquely identifies a target property within the BSON tree. The utility can traverse this path, intelligently handling transitions between BsonDocument and BsonArray containers. For example, a path of {"players", "0", "name"} would navigate to the "players" array, select the first element (index 0), and then access its "name" property.

A key feature is its ability to create nested structures on-the-fly during write operations. If a path specifies a non-existent intermediate document or array, the utility will automatically instantiate and insert it, simplifying data creation logic for consumers.

This class is a foundational component for any system that programmatically modifies BSON data, such as asset editors, configuration managers, or entity data processors.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, BsonTransformationUtil is never instantiated. Its methods are loaded by the JVM ClassLoader and become available for the application's lifetime.
-   **Scope:** Application-wide. Any component with access to the class can invoke its static methods.
-   **Destruction:** The class is unloaded from memory when the application's ClassLoader is garbage collected, typically at shutdown.

## Internal State & Concurrency

-   **State:** BsonTransformationUtil is entirely stateless. It contains no member fields and all operations are pure functions of their inputs. The state that is modified exists exclusively within the BsonDocument instance passed as an argument.

-   **Thread Safety:** The utility itself is thread-safe due to its stateless nature. However, the operations it performs on a given BsonDocument are **not atomic**.

    **WARNING:** Invoking methods from this utility on the *same* BsonDocument instance from multiple threads without external synchronization will lead to race conditions and data corruption. Callers are responsible for ensuring exclusive access to the BSON data structure during mutation.

## API Surface

The public API provides coarse-grained operations for setting, inserting, and removing properties within a BSON structure.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| removeProperty(entity, path) | void | O(L) | Traverses the path and removes the target property. Does nothing if the path does not exist. |
| setProperty(entity, path, value) | void | O(L) | Sets or overwrites the property at the target path. Creates intermediate documents/arrays if they do not exist. |
| insertProperty(entity, path, value) | void | O(L+N) | Inserts the value at the target path. For arrays, this shifts subsequent elements. For documents, it behaves like setProperty. |

*L = Length of the property path. N = Number of elements to shift in an array insertion.*

## Integration Patterns

### Standard Usage

This utility should be used by any service that needs to perform precise modifications to a BSON document without manually writing traversal logic.

```java
// How a developer should normally use this
BsonDocument config = loadConfig(); // Assume this returns a BsonDocument
String[] path = {"renderSettings", "shadows", "quality"};
BsonValue newValue = new BsonString("ULTRA");

// Set the shadow quality property, creating nested objects if necessary
BsonTransformationUtil.setProperty(config, path, newValue);
```

### Anti-Patterns (Do NOT do this)

-   **Concurrent Modification:** Do not call methods of this utility on the same BsonDocument from multiple threads without a lock. This is the most critical anti-pattern and will result in a corrupted BsonDocument.
-   **Invalid Path Segments:** Providing a non-numeric string as a path segment for a BsonArray will result in an IllegalArgumentException. All array indices in the path must be valid integer strings.
-   **Attempted Instantiation:** The class has no public constructor and cannot be instantiated. Do not attempt `new BsonTransformationUtil()`.

## Data Pipeline

BsonTransformationUtil acts as a **Transformer** stage in a data processing pipeline. It does not produce or consume data from I/O sources itself; rather, it operates on in-memory BSON representations.

> Flow:
> BSON Data Source (File, DB, Network) -> BSON Parser -> In-Memory BsonDocument -> **BsonTransformationUtil** (Mutation) -> BSON Serializer -> Data Sink (File, DB, Network)

