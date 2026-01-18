---
description: Architectural reference for FileUtil
---

# FileUtil

**Package:** com.hypixel.hytale.server.core.util.io
**Type:** Utility

## Definition
```java
// Signature
public class FileUtil {
```

## Architecture & Concepts
The **FileUtil** class is a foundational, low-level utility that provides a set of static methods for common file system operations. It resides within the server's core I/O subsystem and serves as a crucial abstraction layer over Java's NIO file APIs.

Its primary architectural role is to centralize and standardize complex or error-prone file manipulations, such as recursive directory deletion and archive extraction. By providing a single, authoritative implementation for these tasks, it ensures consistency, improves security, and reduces boilerplate code across the server codebase.

A key design feature is its security-conscious approach. The **unzipFile** method, for example, contains an explicit check to prevent "Zip Slip" path traversal vulnerabilities, making it a safe boundary for processing potentially untrusted user-generated content like mods or world saves.

## Lifecycle & Ownership
- **Creation:** As a static utility class, **FileUtil** is never instantiated. Its methods are invoked directly on the class itself.
- **Scope:** The class is loaded by the JVM's ClassLoader and its methods are available for the entire application lifetime. It has no instance-specific scope.
- **Destruction:** The class is unloaded when the server process terminates. There is no instance state to manage or clean up.

## Internal State & Concurrency
- **State:** **Stateless and Immutable**. **FileUtil** contains no instance fields and its static fields are all declared final. All operations are pure functions with respect to the class itself; their outcomes depend solely on the input arguments and the state of the external file system.

- **Thread Safety:** **Conditionally Thread-Safe**. The methods are re-entrant and do not share internal state. However, they operate on the file system, which is a shared, mutable, and external resource. Therefore, the class itself is thread-safe, but the operations it performs are not atomic.

    **WARNING:** Callers are responsible for implementing their own synchronization strategies. Invoking **FileUtil** methods on the same or overlapping file paths from multiple threads without external locking will lead to race conditions, data corruption, and unpredictable `IOExceptions`.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| unzipFile(path, buffer, zipStream, zipEntry, name) | void | O(N) | Unpacks a single entry from a zip stream. Throws `ZipException` if a path traversal attack is detected. N is the size of the entry. |
| copyDirectory(origin, destination) | void | O(N) | Recursively copies an entire directory structure. N is the number of nodes in the directory tree. |
| moveDirectoryContents(origin, destination, options) | void | O(N) | Recursively moves the contents of a directory to a new location. The origin directory itself is not moved. N is the number of nodes. |
| deleteDirectory(path) | void | O(N) | Recursively deletes a directory and all its contents. This operation is irreversible. N is the number of nodes. |

## Integration Patterns

### Standard Usage
**FileUtil** should be used for any non-trivial file system manipulation, especially those involving directory trees or archives. It is the preferred mechanism for setup, cleanup, and data migration tasks.

```java
// Example: Safely unpacking an archive and cleaning up afterwards
Path destination = Paths.get("./server/mods/unpacked/my-mod");
Path archive = Paths.get("./downloads/my-mod.zip");

try (ZipInputStream zis = new ZipInputStream(Files.newInputStream(archive))) {
    byte[] buffer = new byte[4096];
    ZipEntry entry;
    while ((entry = zis.getNextEntry()) != null) {
        // Use the safe unzip method to prevent path traversal
        FileUtil.unzipFile(destination, buffer, zis, entry, entry.getName());
    }
} catch (IOException e) {
    // If anything fails, perform a robust cleanup
    if (Files.exists(destination)) {
        FileUtil.deleteDirectory(destination);
    }
    // Handle or re-throw exception
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring IOExceptions:** The methods in this class throw `IOException` for a reason. Blindly catching and ignoring these exceptions can leave the file system in a corrupt or indeterminate state.
- **Concurrent File System Access:** Do not call **copyDirectory** on a path while another thread calls **deleteDirectory** on the same path. This will cause a race condition and likely result in a `NoSuchFileException` or partial data. All concurrent operations on a shared path must be externally synchronized.
- **Unchecked Exception Handling:** The **deleteDirectory** method uses **SneakyThrow** to convert a checked `IOException` into an unchecked `RuntimeException`. Callers must be aware of this and be prepared to catch this runtime exception, as it indicates a failure during the deletion process.

## Data Pipeline
**FileUtil** acts as a processing stage in various I/O pipelines, but it does not manage a pipeline itself. The most representative data flow is within the **unzipFile** method.

> Flow:
> `ZipInputStream` (Compressed Byte Stream) -> **FileUtil.unzipFile** (Decompression & Path Validation) -> `OutputStream` (Raw Bytes) -> File System (Data on Disk)

