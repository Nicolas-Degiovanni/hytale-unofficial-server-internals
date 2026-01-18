---
description: Architectural reference for AssetPathUtil
---

# AssetPathUtil

**Package:** com.hypixel.hytale.builtin.asseteditor.util
**Type:** Utility

## Definition
```java
// Signature
public class AssetPathUtil {
```

## Architecture & Concepts
AssetPathUtil is a stateless, static utility class that serves as the single source of truth for asset path and filename validation within the Hytale engine. Its primary function is to enforce a consistent set of naming conventions, ensuring that assets created or managed by the engine are compatible across different filesystems, with a particular focus on Windows platform constraints.

This class acts as a foundational service for higher-level systems such as the AssetManager, AssetRepository, and any UI components within the Asset Editor. By centralizing this logic, it prevents disparate parts of the application from implementing their own, potentially conflicting, validation rules. The rules enforced—such as disallowing control characters, specific symbols, and reserved system filenames like CON or PRN—are critical for preventing filesystem errors and maintaining engine stability.

## Lifecycle & Ownership
- **Creation:** As a class containing only static members, AssetPathUtil is never instantiated. It is loaded into the JVM by the ClassLoader when one of its methods or constants is first referenced.
- **Scope:** The class and its methods are available globally for the entire application lifetime once loaded.
- **Destruction:** The class is unloaded from memory only when the JVM shuts down. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** AssetPathUtil is completely **stateless**. It contains no mutable fields. Its internal constants, such as the regex pattern and the list of reserved names, are immutable and defined at compile time.
- **Thread Safety:** This class is inherently **thread-safe**. All its methods are pure functions that operate exclusively on their input arguments without causing side effects. It can be safely invoked from any thread without external locking or synchronization mechanisms.

## API Surface
The public API consists of static methods for path validation and manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isInvalidFileName(Path path) | boolean | O(N) | Validates a path's filename against engine rules. Returns true if the name is empty, ends with a period, contains illegal characters, or matches a reserved OS filename. N is the length of the filename. |
| removeInvalidFileNameChars(String name) | String | O(N) | Sanitizes a string by removing all characters that are invalid in a filename according to the engine's rules. N is the length of the input string. |
| getIdFromPath(Path path) | String | O(N) | Extracts the logical asset ID from a path by returning the filename without its final file extension. N is the length of the path's string representation. |

## Integration Patterns

### Standard Usage
This utility should be used by any system component before it attempts to perform a filesystem write operation or when it needs to derive a logical ID from a file path.

```java
// Example: Validating a user-provided asset name before creation
Path targetPath = Paths.get("assets", "models", "My New Model.json");

if (AssetPathUtil.isInvalidFileName(targetPath)) {
    // Abort the operation and provide feedback to the user
    throw new IllegalArgumentException("The provided asset name is invalid.");
} else {
    // Proceed with writing the asset to the filesystem
    assetWriter.save(targetPath, assetData);
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Validation:** Do not implement separate or alternative filename validation logic elsewhere in the codebase. This leads to rule fragmentation and inconsistent behavior. All filename validation must delegate to this class.
- **Assuming Global Uniqueness:** The string returned by getIdFromPath is derived purely from the filename. Do not assume this ID is globally unique across the entire project without further qualification by a higher-level asset management system.

## Data Pipeline
AssetPathUtil is not a pipeline stage itself, but rather a gatekeeper or transformer used by components within a larger data flow. It ensures that data representing filenames conforms to system requirements before being passed to the next stage, such as the filesystem.

> **Sanitization Flow:**
> User Input String -> **AssetPathUtil.removeInvalidFileNameChars()** -> Sanitized Filename -> Filesystem Write API

> **Validation Flow:**
> Path Object -> **AssetPathUtil.isInvalidFileName()** -> Boolean Gate -> (Allow/Deny) Filesystem Operation

> **ID Extraction Flow:**
> Filesystem Path -> **AssetPathUtil.getIdFromPath()** -> Asset ID String -> Asset Cache Lookup

