---
description: Architectural reference for PathUtil
---

# PathUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class PathUtil {
```

## Architecture & Concepts
PathUtil is a stateless, static utility class that serves as the engine's central authority for file system path manipulation. Its primary architectural role is to create a normalized, OS-agnostic abstraction over the `java.nio.file.Path` API, shielding the rest of the application from platform-specific inconsistencies (e.g., Windows backslashes vs. Unix forward slashes, drive letters).

This class is a foundational dependency for any system that interacts with the file system, including the asset pipeline, configuration loaders, logging services, and user data managers. It provides robust, predictable methods for common but error-prone operations such as resolving paths relative to the user's home directory (tilde expansion), safely calculating relative paths, and verifying path containment. By centralizing this logic, PathUtil prevents scattered, duplicated, and potentially insecure path handling code throughout the engine.

## Lifecycle & Ownership
- **Creation:** As a static utility class, PathUtil is never instantiated. Its bytecode is loaded by the JVM ClassLoader upon the first static access to any of its methods or fields.
- **Scope:** The class and its static methods are available for the entire application lifetime.
- **Destruction:** The class is unloaded when the application's ClassLoader is garbage collected, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** PathUtil is **stateless**. It contains no mutable fields. The single static field, PATH_PATTERN, is a `java.util.regex.Pattern` object, which is immutable and thread-safe by design. All methods are pure functions that operate exclusively on their input arguments.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its stateless nature, all public methods can be invoked concurrently from any thread without requiring external synchronization or locks.

## API Surface
The API provides a set of primitives for path resolution, transformation, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(String path) | Path | O(L) | Resolves a string into a normalized Path. Critically, this handles tilde (`~`) expansion to the user's home directory. |
| getParent(Path path) | Path | O(L) | Safely retrieves the parent directory of a path, correctly handling both absolute and relative path edge cases. |
| relativize(Path pathA, Path pathB) | Path | O(L) | Calculates the relative path from pathA to pathB. Includes guards against different file system roots (e.g., C: vs D:). |
| isChildOf(Path parent, Path child) | boolean | O(L) | Performs a normalized, absolute path check to determine if the child path is located within the parent directory tree. |
| forEachParent(Path path, Path limit, Consumer consumer) | void | O(D) | Traverses up the directory hierarchy from a starting path, invoking a consumer for each parent until an optional limit is reached. D is the depth. |
| toUnixPathString(Path path) | String | O(L) | Converts a Path to a String using forward slashes as separators, ensuring a consistent format for serialization or network transfer. |

*L = Length of the path string, D = Directory depth*

## Integration Patterns

### Standard Usage
PathUtil should be the default tool for resolving any file path that originates from configuration, user input, or any other external source. The primary entry point is typically the `get` method for initial resolution and normalization.

```java
// Correctly resolve a path to a user-specific configuration file
// The get() method automatically handles the "~/" prefix.
Path configDir = PathUtil.get("~/Library/Application Support/Hytale");
Path configFile = configDir.resolve("game.json");

// Use this normalized path with file system services.
if (Files.exists(configFile)) {
    // ... load configuration
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Tilde Expansion:** Do not manually check for `~` and replace it with `System.getProperty("user.home")`. This bypasses the normalization and validation logic built into `PathUtil.get`, leading to brittle code.
    ```java
    // INCORRECT: Bypasses PathUtil's normalization logic
    String userPath = "~/data";
    if (userPath.startsWith("~")) {
        userPath = System.getProperty("user.home") + userPath.substring(1);
    }
    Path path = Paths.get(userPath); // This path may not be correctly normalized
    ```

- **Assuming Path Separators:** Never construct paths by concatenating strings with hardcoded `/` or `\` separators. This is a primary source of cross-platform bugs. Always use `Path.resolve()` on objects returned from PathUtil.

## Data Pipeline
PathUtil acts as a critical sanitization and normalization stage at the beginning of any data pipeline involving file system paths. It transforms unreliable string inputs into reliable, structured Path objects.

> Flow:
> Raw String Path (e.g., from config file) -> **PathUtil.get()** -> Normalized `java.nio.file.Path` Object -> AssetManager / FileManager -> File I/O Operation

