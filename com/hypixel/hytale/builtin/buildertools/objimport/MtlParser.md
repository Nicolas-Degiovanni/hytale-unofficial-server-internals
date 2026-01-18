---
description: Architectural reference for MtlParser
---

# MtlParser

**Package:** com.hypixel.hytale.builtin.buildertools.objimport
**Type:** Utility

## Definition
```java
// Signature
public final class MtlParser {
    // Note: The constructor is private. This class cannot be instantiated.
}
```

## Architecture & Concepts
The MtlParser is a stateless, file-format-specific parser responsible for interpreting Wavefront MTL (.mtl) material definition files. Its sole function is to translate the on-disk, text-based representation of 3D model materials into a structured, in-memory map of MtlMaterial records.

This class serves as a critical dependency for the ObjImporter system. While the ObjImporter handles the geometry (vertices, normals, faces) from an .obj file, the MtlParser provides the corresponding surface appearance data (diffuse colors and texture paths). It operates as a leaf component in the asset import pipeline, having no dependencies on other engine systems beyond standard Java I/O libraries. Its design prioritizes simplicity and correctness for a narrowly defined parsing task.

## Lifecycle & Ownership
-   **Creation:** The MtlParser is a utility class with a private constructor and is therefore **never instantiated**. All of its methods are static and are invoked directly on the class.
-   **Scope:** The class is loaded by the JVM's ClassLoader and its static methods are available for the entire application lifetime. Any operation performed by the parser, such as the `parse` method, is transient. The state and resources associated with a parsing operation exist only for the duration of that single method call.
-   **Destruction:** Not applicable. The class is unloaded when the application terminates. The `parse` method uses a try-with-resources block to manage the `BufferedReader`, ensuring that file handles are deterministically and safely closed upon completion or if an exception occurs.

## Internal State & Concurrency
-   **State:** The MtlParser class is **stateless**. The primary `parse` method initializes and mutates local variables confined to its own stack frame. This state is discarded upon method exit. The returned `MtlMaterial` objects are Java records, making them inherently immutable.
-   **Thread Safety:** This class is **thread-safe**. The `parse` method is static and fully re-entrant. Multiple threads can invoke `parse` concurrently without risk of interference, as each call operates on its own set of local variables.

    **Warning:** While the parser itself is thread-safe, the underlying file system is not. Concurrent calls to `parse` on the same `Path` while another process is modifying the file will lead to undefined behavior. The caller is responsible for ensuring file access is properly synchronized at a higher level if such scenarios are possible.

## API Surface
The public contract is minimal, consisting of a single static factory method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(Path path) | Map | O(L) | Parses the specified .mtl file. L is the number of lines in the file. Throws IOException if the file cannot be read. |
| MtlMaterial.getDiffuseColorRGB() | int[] | O(1) | A method on the returned data record. Converts the float-based diffuse color to an integer-based RGB array. Returns null if no color is defined. |

## Integration Patterns

### Standard Usage
The MtlParser is not intended for direct use by most game systems. It is designed to be invoked by higher-level asset importers, primarily the ObjImporter, which coordinates the loading of both geometry and material data.

```java
// Typically invoked by a higher-level importer
Path materialFilePath = Paths.get("assets/models/props/chest.mtl");
Map<String, MtlParser.MtlMaterial> materials;

try {
    materials = MtlParser.parse(materialFilePath);
    // The 'materials' map is now ready to be used by the ObjImporter
    // to link material names to their properties.
} catch (IOException e) {
    // Handle file not found or read errors
    log.error("Failed to parse material library: " + materialFilePath, e);
}
```

### Anti-Patterns (Do NOT do this)
-   **Attempted Instantiation:** Do not attempt to create an instance of MtlParser using reflection. The class is designed to be a static utility.
-   **Ignoring IOExceptions:** The `parse` method is a file I/O operation and is explicitly declared to throw IOException. Failure to catch this exception will result in unhandled exceptions and potential application instability. Always wrap calls in a try-catch block.
-   **Assuming Data Completeness:** Do not assume that a parsed MtlMaterial will contain non-null values for both `diffuseColor` and `diffuseTexturePath`. The .mtl file format allows for materials to define one, both, or neither. Always perform null checks on the fields of the returned MtlMaterial records.

## Data Pipeline
The MtlParser acts as the entry point for material data into the model loading system. It transforms raw text data from the file system into a structured, queryable map.

> Flow:
> .mtl File on Disk -> `java.nio.file.Path` -> **MtlParser.parse()** -> `Map<String, MtlMaterial>` -> ObjImporter / Asset Processor

