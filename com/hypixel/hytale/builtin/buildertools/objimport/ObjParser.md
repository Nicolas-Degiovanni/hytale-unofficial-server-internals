---
description: Architectural reference for ObjParser
---

# ObjParser

**Package:** com.hypixel.hytale.builtin.buildertools.objimport
**Type:** Utility

## Definition
```java
// Signature
public final class ObjParser {
```

## Architecture & Concepts
The ObjParser is a stateless, single-purpose utility class designed for the translation of the Wavefront OBJ 3D model format into an engine-consumable in-memory representation. It serves as a critical component of the asset ingestion pipeline, primarily within the context of developer tools and modding workflows.

Its core responsibility is to perform a direct, one-shot conversion of an OBJ file specified by a file path into a structured ObjMesh record. The parser is intentionally designed to be self-contained, with no dependencies on other engine systems like the AssetManager or rendering pipeline. This decoupling ensures it can be used in isolated, offline tools or early in the engine bootstrap process before higher-level systems are available.

The parser handles key features of the OBJ specification, including:
*   Vertex positions (v)
*   Texture coordinates (vt)
*   Polygonal faces (f), with automatic triangulation of quads and n-gons.
*   Material library references (mtllib)
*   Material usage declarations (usemtl)

Error handling is explicit. The component will throw a checked ObjParseException for any syntactical or structural violations within the source file, forcing the calling system to handle malformed data gracefully.

## Lifecycle & Ownership
-   **Creation:** The ObjParser is a final class with a private constructor. It is a pure utility and **cannot be instantiated**. All operations are exposed via static methods.
-   **Scope:** The class and its static methods are available for the entire application lifetime once loaded by the JVM ClassLoader. It holds no state and therefore has no instance-level scope.
-   **Destruction:** Not applicable. The class is unloaded when its ClassLoader is garbage collected. The ObjMesh object returned by the parse method is a transient data container whose lifecycle is entirely managed by the caller. The caller assumes full ownership of the returned ObjMesh and is responsible for its eventual garbage collection.

## Internal State & Concurrency
-   **State:** The ObjParser class is **stateless**. The primary parse method operates exclusively on local variables created within its own stack frame. All state required for parsing (e.g., lists of vertices, faces) is created, populated, and encapsulated into the returned ObjMesh object before the method returns. The returned ObjMesh record, however, contains mutable collections and is **not immutable**.

-   **Thread Safety:** The static parse method is **thread-safe**. Multiple threads can invoke ObjParser.parse concurrently on different or even the same file paths without risk of data corruption or race conditions. This is guaranteed by its stateless design.

    **WARNING:** The returned ObjMesh object is **not thread-safe**. Its internal lists are mutable. If an ObjMesh instance is shared between threads, the client code is responsible for implementing external synchronization to prevent concurrent modification exceptions or inconsistent state.

## API Surface
The public contract is minimal, consisting of a single static factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(Path path) | ObjMesh | O(L) | Parses the specified OBJ file. L is the number of lines in the file. Throws IOException on file read errors and ObjParseException on format violations. |

## Integration Patterns

### Standard Usage
The parser should be invoked directly to load model data from the file system. The caller is responsible for handling both I/O and parsing exceptions.

```java
// How a developer should normally use this
import java.nio.file.Paths;

Path modelPath = Paths.get("assets/models/character.obj");
ObjParser.ObjMesh mesh;

try {
    mesh = ObjParser.parse(modelPath);
    // The mesh object is now ready for processing by a mesh builder,
    // renderer, or other game system.
    System.out.println("Successfully parsed " + mesh.vertices().size() + " vertices.");
} catch (IOException e) {
    // Handle file system access errors (e.g., file not found)
    log.error("Failed to read model file: " + modelPath, e);
} catch (ObjParser.ObjParseException e) {
    // Handle malformed OBJ data
    log.error("Failed to parse model file due to invalid format: " + modelPath, e);
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Exceptions:** The parse method throws checked exceptions. Failing to catch IOException and ObjParseException will result in unhandled exceptions and application instability. Always wrap calls in a try-catch block.
-   **Concurrent Modification:** Do not pass the returned ObjMesh object to multiple threads that may modify it (e.g., by calling transformZUpToYUp) without proper locking. This will lead to unpredictable behavior and race conditions.
-   **Assuming Data Presence:** Do not assume the returned ObjMesh contains UVs or materials. Always use the provided helper methods like hasUvCoordinates() and hasMaterials() to check for the presence of optional data before attempting to access it.

## Data Pipeline
The ObjParser functions as a self-contained data transformation stage. It does not interact with event buses or other systems; it simply converts data from one format to another.

> Flow:
> File System (Wavefront .obj file) -> `java.nio.file.Path` -> **ObjParser.parse()** -> `ObjParser.ObjMesh` (In-Memory Representation) -> Engine Mesh Builder / Physics Collider Generator

