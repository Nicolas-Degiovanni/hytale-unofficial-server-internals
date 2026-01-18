---
description: Architectural reference for PrefabBufferValidator
---

# PrefabBufferValidator

**Package:** com.hypixel.hytale.builtin.blockphysics
**Type:** Utility

## Definition
```java
// Signature
public class PrefabBufferValidator {
```

## Architecture & Concepts
The PrefabBufferValidator is a stateless, static utility class that serves as a critical component of the server's content integrity pipeline. Its primary function is to perform deep, structural validation of Hytale prefab files (`.prefab.json`), ensuring they conform to engine requirements and will not cause unexpected runtime behavior.

This class acts as a high-level orchestrator for prefab validation. It does not contain the validation logic for every possible component itself. Instead, it delegates specific checks to specialized subsystems:
*   **WorldValidationUtil:** Handles foundational checks for block states and entity component validity.
*   **FillerBlockUtil:** Manages the complex rules governing filler blocks, which are essential for procedural world generation and terrain blending.

PrefabBufferValidator is designed for use in offline build tools, server startup diagnostics, and developer-facing validation commands. By centralizing the validation entry points, it provides a consistent and reliable mechanism for catching content errors before they reach a production environment.

## Lifecycle & Ownership
- **Creation:** As a static utility class, PrefabBufferValidator is never instantiated. The Java ClassLoader loads it into memory on first access.
- **Scope:** The class is available for the entire lifetime of the Java Virtual Machine.
- **Destruction:** The class is unloaded from memory when the JVM shuts down. There is no instance-level state to manage or clean up.

## Internal State & Concurrency
- **State:** The PrefabBufferValidator is **stateless and immutable**. It holds no mutable static or instance fields, and its operations are purely functional. Each method call operates exclusively on the arguments provided to it.
- **Thread Safety:** This class is **unconditionally thread-safe**. Its stateless nature ensures that concurrent calls from multiple threads cannot interfere with one another. However, callers should be aware that high-volume validation operations, particularly those involving filesystem access like validateAllPrefabs, can lead to I/O contention.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validateAllPrefabs(List) | List&lt;String&gt; | O(N * M) | High-level entry point. Scans all standard prefab directories (WorldGen, Asset, Server) and returns a consolidated list of validation errors. |
| validatePrefabsInPath(Path, Set) | List&lt;String&gt; | O(N * M) | Scans a user-specified directory tree for prefab files and validates them. Throws an unchecked exception on I/O failure. |
| validate(IPrefabBuffer, Set) | String | O(M) | Performs validation on a single, in-memory IPrefabBuffer. Returns a formatted string of errors or null if the prefab is valid. |

*Complexity Note: N represents the number of prefab files, and M represents the average number of blocks and entities within a single prefab.*

## Integration Patterns

### Standard Usage
The most common use case is to validate all known game prefabs during a build step or server health check. This is achieved by calling the static validateAllPrefabs method.

```java
// Example: Running a full server content validation
List<ValidationOption> options = List.of(
    ValidationOption.BLOCKS,
    ValidationOption.ENTITIES,
    ValidationOption.BLOCK_FILLER
);

List<String> allErrors = PrefabBufferValidator.validateAllPrefabs(options);

if (allErrors.isEmpty()) {
    System.out.println("All prefabs validated successfully.");
} else {
    System.err.println("Prefab validation failed:");
    allErrors.forEach(System.err::println);
}
```

### Anti-Patterns (Do NOT do this)
- **Mismanaging Buffer Lifecycle:** The `validate` method operates on a raw IPrefabBuffer. The caller is responsible for managing the buffer's lifecycle. Failure to release the buffer will result in a memory leak.

    ```java
    // BAD: Leaks the IPrefabBuffer resource
    IPrefabBuffer prefab = PrefabBufferUtil.getCached(path);
    String errors = PrefabBufferValidator.validate(prefab, options);

    // GOOD: Ensures the buffer is always released
    IPrefabBuffer prefab = PrefabBufferUtil.getCached(path);
    try {
        String errors = PrefabBufferValidator.validate(prefab, options);
        // ... process errors
    } finally {
        prefab.release();
    }
    ```

- **Ignoring I/O Exceptions:** The methods that scan the filesystem wrap IOException in an unchecked SneakyThrow. Production-grade tooling must be prepared to catch these exceptions to handle cases like missing directories or permission errors.

## Data Pipeline
The validator sits at the end of the prefab loading process, acting as a quality gate before the prefab data is used by other engine systems.

> Flow:
> Filesystem (`.prefab.json`) -> `PrefabBufferUtil` (Deserialization & Caching) -> `IPrefabBuffer` (In-Memory Representation) -> **PrefabBufferValidator** (Analysis & Error Detection) -> `List<String>` (Structured Error Report)

