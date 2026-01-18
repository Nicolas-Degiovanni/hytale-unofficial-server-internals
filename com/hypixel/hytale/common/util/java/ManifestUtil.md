---
description: Architectural reference for ManifestUtil
---

# ManifestUtil

**Package:** com.hypixel.hytale.common.util.java
**Type:** Utility

## Definition
```java
// Signature
public class ManifestUtil {
```

## Architecture & Concepts
ManifestUtil is a static utility class that serves as the primary, runtime interface to build-time metadata embedded within the application's JAR file. Its fundamental purpose is to extract versioning, revision, and vendor information from the `META-INF/MANIFEST.MF` file.

The core architectural pattern employed is lazy initialization and caching via the **CachedSupplier** mechanism. This design ensures that the potentially expensive I/O operation of scanning the classpath, locating the correct manifest, and parsing it occurs only once during the application's lifecycle. The first time any method in this class is called, the `MANIFEST` supplier is resolved. It iterates through all `MANIFEST.MF` files available to the classloader, specifically identifying the official Hytale manifest by its **Implementation-Vendor-Id** attribute, which must be `com.hypixel.hytale`.

Subsequent calls to any method in this class will return the cached values instantly without incurring any I/O or parsing overhead. This makes the class highly performant for frequent version checks, such as in logging, diagnostics, or network handshakes.

## Lifecycle & Ownership
- **Creation:** As a static utility class, ManifestUtil is never instantiated. Its static fields are initialized by the JVM class loader when the class is first referenced in the code. The internal state, held by the CachedSupplier instances, is populated upon the first invocation of any public method.
- **Scope:** The cached manifest data is static and global. It persists for the entire lifetime of the application, bound to the class loader.
- **Destruction:** The cached data is eligible for garbage collection only when the application's class loader is unloaded, which typically occurs at JVM shutdown. There is no manual cleanup mechanism.

## Internal State & Concurrency
- **State:** The internal state is the parsed Manifest object and its derived string values (version, revision, etc.). This state is held within static, final CachedSupplier fields. After the initial computation, this state is **effectively immutable**.
- **Thread Safety:** This class is **thread-safe**. The underlying CachedSupplier implementation guarantees that the manifest loading and parsing logic will be executed by only one thread. Any other threads attempting to access the data concurrently will block until the initial computation is complete and then receive the same cached result. All subsequent reads are non-blocking and safe.

## API Surface
The complexity of the first call to any method is O(N), where N is the number of manifest files on the classpath. All subsequent calls are O(1).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isJar() | boolean | O(1) | Returns true if a valid Hytale manifest was found and loaded. |
| getManifest() | Manifest | O(1) | Returns the raw, parsed Manifest object. Returns null if not running from a JAR or if the manifest is not found. |
| getImplementationVersion() | String | O(1) | Returns the `Implementation-Version` string. Fallback is "NoJar" or "UNKNOWN". |
| getVersion() | Semver | O(1) | Returns the parsed semantic version object. Returns null if the version string is invalid or not found. |
| getImplementationRevisionId() | String | O(1) | Returns the `Implementation-Revision-Id` string, typically a Git commit hash. Fallback is "NoJar" or "UNKNOWN". |
| getPatchline() | String | O(1) | Returns the `Implementation-Patchline` string. Fallback is "dev". |

## Integration Patterns

### Standard Usage
This is a static utility class. Methods should be called directly without instantiation. It is commonly used during application startup to log version information or by network systems to report the client version.

```java
// Log the application version and revision at startup
import com.hypixel.hytale.logger.HytaleLogger;

HytaleLogger logger = HytaleLogger.getLogger();
String version = ManifestUtil.getImplementationVersion();
String revision = ManifestUtil.getImplementationRevisionId();

logger.info("Starting Hytale Client version: " + version + " (rev: " + revision + ")");
```

### Anti-Patterns (Do NOT do this)
- **Assuming Dynamic Updates:** Do not poll methods from this class expecting the values to change during runtime. The manifest is read once and cached for the application's entire lifetime.
- **Ignoring Nulls and Fallbacks:** Do not assume methods like getVersion will always return a valid Semver object. In a development environment (e.g., running from an IDE) or a corrupted build, the manifest may be absent. The code will return null or a fallback string like "NoJar". Always perform null checks or handle the fallback cases appropriately.

## Data Pipeline
ManifestUtil acts as a data source, not a processing node in a larger pipeline. It originates versioning information that is consumed by other systems.

> Flow:
> JAR File System (`META-INF/MANIFEST.MF`) -> JVM ClassLoader -> **ManifestUtil (Scan, Parse, Cache)** -> Consuming System (e.g., Logger, Crash Reporter, Network Client)

