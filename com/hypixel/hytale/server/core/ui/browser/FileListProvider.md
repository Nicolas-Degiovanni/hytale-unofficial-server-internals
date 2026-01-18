---
description: Architectural reference for FileListProvider
---

# FileListProvider

**Package:** com.hypixel.hytale.server.core.ui.browser
**Type:** Strategy Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface FileListProvider {
   @Nonnull
   List<FileListProvider.FileEntry> getFiles(@Nonnull Path var1, @Nonnull String var2);

   public record FileEntry(@Nonnull String name, @Nonnull String displayName, boolean isDirectory, int matchScore) {
      // Constructors...
   }
}
```

## Architecture & Concepts
The FileListProvider interface defines a behavioral contract for components that need to enumerate file-like resources. As a functional interface, its primary role is to facilitate a **Strategy Pattern**, decoupling user interface components (like a file browser) from the underlying data source.

This abstraction allows the UI to request a list of files from a given path without any knowledge of the storage medium. Implementations can source files from the local disk, a virtual file system (VFS) for mods, a network location, or any other hierarchical data store. The interface's single method, getFiles, is designed to support real-time filtering, as indicated by the filter string parameter.

The nested FileEntry record acts as a standardized Data Transfer Object (DTO), ensuring that all providers return data in a consistent, immutable format that the UI can readily consume.

### Lifecycle & Ownership
As an interface, FileListProvider does not have its own lifecycle. The following describes the lifecycle of a typical *implementation* of this interface.

- **Creation:** An implementation is instantiated by a higher-level system responsible for managing the data source. For example, a `LocalDiskProvider` would be created by a server's file management service, while a `ModVFSProvider` would be created by the mod loading system. The instance is then injected into the UI component that requires it.
- **Scope:** The lifetime of a provider instance is tied to its consumer. It typically persists as long as the UI view that uses it is active. It is not a global singleton.
- **Destruction:** The provider object is eligible for garbage collection when the consuming UI component is destroyed and all references to it are released. No explicit cleanup method is defined in the contract.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Implementations, however, may be stateful. For example, a provider might cache directory listings to improve performance for repeated requests. Callers must not assume that providers are stateless.
- **Thread Safety:** The interface contract makes no guarantees about thread safety. Implementations are **not** required to be thread-safe. It is the responsibility of the consuming system to ensure that calls to getFiles are synchronized or executed on an appropriate thread (e.g., a dedicated I/O worker thread) to prevent blocking the main server or UI thread.

**Warning:** Executing a file system-backed implementation of getFiles on a performance-critical thread can lead to severe application stalls and unresponsiveness.

## API Surface
The public contract consists of a single method and a nested record.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFiles(Path, String) | List<FileEntry> | I/O Bound, O(N) | Retrieves a filtered list of file entries from the specified path. N is the number of items in the source directory. Throws an exception on I/O errors. |
| FileEntry | record | O(1) | An immutable DTO representing a single file or directory entry. |

## Integration Patterns

### Standard Usage
The FileListProvider is intended to be used as a strategy, often supplied via dependency injection or as a lambda expression. The consumer invokes getFiles to refresh its view based on user input.

```java
// A UI component receives a provider instance and uses it to populate a list.
public class FileBrowserUI {
    private final FileListProvider fileProvider;
    private Path currentPath;

    public FileBrowserUI(FileListProvider fileProvider) {
        this.fileProvider = fileProvider;
    }

    public void onUserSearch(String filter) {
        // Potentially offload this call to a worker thread
        List<FileListProvider.FileEntry> entries = fileProvider.getFiles(currentPath, filter);
        this.updateUI(entries);
    }
}

// Example instantiation with a lambda for a simple, in-memory provider.
FileListProvider memoryProvider = (path, filter) -> List.of(
    new FileListProvider.FileEntry("save1.json", false),
    new FileListProvider.FileEntry("config.toml", false)
);
FileBrowserUI browser = new FileBrowserUI(memoryProvider);
```

### Anti-Patterns (Do NOT do this)
- **Blocking the Main Thread:** Do not call getFiles from an implementation that performs disk I/O directly on the main game loop or UI thread. This will cause the application to freeze.
- **Implementation-Specific Logic:** The consumer should never attempt to cast the provider to a concrete type to access implementation-specific methods. This violates the abstraction and creates a rigid, brittle coupling.
- **Assuming Statelessness:** Do not assume that two consecutive calls to getFiles with the same arguments will return identical results instantaneously. The underlying file system may have changed, or the provider may have caching logic.

## Data Pipeline
The data flow for this component is typically initiated by a user action and terminates with a UI update.

> Flow:
> User Input (e.g., search text) -> UI Event Handler -> **FileListProvider.getFiles()** -> Underlying Storage (Disk, VFS) -> List<FileEntry> -> UI Rendering Logic

