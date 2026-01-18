---
description: Architectural reference for BackupUtil
---

# BackupUtil

**Package:** com.hypixel.hytale.server.core.util.backup
**Type:** Utility

## Definition
```java
// Signature
class BackupUtil {
```

## Architecture & Concepts
BackupUtil is a package-private, stateless utility class that provides the fundamental I/O and communication primitives for the server's world backup system. It is not a self-contained service but rather a collection of low-level, static helper methods designed to be orchestrated by a higher-level manager, such as a scheduled backup task.

The class serves three primary functions:
1.  **Archiving:** It performs the core operation of walking a world directory and creating an uncompressed Zip archive. The choice to use zero compression (`setLevel(0)`) is a deliberate design trade-off, prioritizing backup speed and minimizing CPU load over final archive size.
2.  **Cleanup:** It provides logic for identifying and listing old backups to facilitate automated backup rotation.
3.  **Communication:** It acts as a bridge to the global server state (Universe) and permissions system (PermissionsModule) to broadcast backup status and error messages to connected clients.

This utility directly couples file system operations with server-wide communication, making it a critical component that sits between raw I/O and the player-facing notification system.

### Lifecycle & Ownership
-   **Creation:** As a utility class containing only static methods, BackupUtil is never instantiated. The Java ClassLoader loads it into memory on first use.
-   **Scope:** The class and its methods are available for the entire lifetime of the server application.
-   **Destruction:** The class is unloaded when the Java Virtual Machine shuts down. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** BackupUtil is entirely **stateless**. It contains no member fields and all operations are performed on data passed via method arguments. Each method invocation is independent and produces no side effects on the class itself.

-   **Thread Safety:** The methods are internally thread-safe as they do not share mutable state. However, the operations they perform on external systems are **not atomic** and require careful management by the caller.
    -   **WARNING:** The `walkFileTreeAndZip` and `findOldBackups` methods perform blocking, multi-step file I/O operations. They are not safe to call concurrently on the same directory paths. The caller is responsible for ensuring that these methods are invoked from a dedicated background thread to prevent freezing the main server loop.
    -   **WARNING:** The broadcast methods access the global `Universe` singleton. Their thread safety is entirely dependent on the guarantees provided by the Universe and its underlying components. Calls must be synchronized if the underlying collections are not concurrent-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| walkFileTreeAndZip(Path, Path) | void | I/O Bound | Reads all files in `sourceDir` and writes them to an uncompressed Zip archive at `zipPath`. Throws IOException on failure. |
| broadcastBackupStatus(boolean) | void | O(N) | Broadcasts a `WorldSavingStatus` packet to all N players connected to the Universe. |
| broadcastBackupError(Throwable) | void | O(N) | Sends a formatted error message to all N connected players who have the `hytale.status.backup.error` permission. |
| findOldBackups(Path, int) | List<Path> | O(F log F) | Scans a directory for Zip files, sorts them by creation time, and returns a list of the oldest files that exceed the `maxBackupCount`. F is the number of files. Returns null if the directory does not exist or if no old backups are found. |

## Integration Patterns

### Standard Usage
BackupUtil is intended to be used by a dedicated backup orchestration service. The orchestrator is responsible for scheduling, threading, and error handling.

```java
// Executed on a dedicated backup thread
Path worldDir = server.getWorldDirectory();
Path backupDir = server.getBackupDirectory();
Path newBackupFile = backupDir.resolve("backup-" + System.currentTimeMillis() + ".zip");

try {
    // 1. Notify players that a backup is starting
    BackupUtil.broadcastBackupStatus(true);

    // 2. Perform the blocking I/O operation
    BackupUtil.walkFileTreeAndZip(worldDir, newBackupFile);

    // 3. Clean up old backups
    List<Path> oldBackups = BackupUtil.findOldBackups(backupDir, MAX_BACKUPS);
    if (oldBackups != null) {
        for (Path old : oldBackups) {
            Files.delete(old);
        }
    }

} catch (IOException e) {
    // 4. Notify privileged players of an error
    BackupUtil.broadcastBackupError(e);

} finally {
    // 5. Notify players that the backup is complete
    BackupUtil.broadcastBackupStatus(false);
}
```

### Anti-Patterns (Do NOT do this)
-   **Calling from Main Server Thread:** The `walkFileTreeAndZip` method is a long-running, blocking I/O operation. Invoking it from the main game loop will cause severe server lag and unresponsiveness. **All file operations must be delegated to a worker thread.**
-   **Ignoring Universe State:** The broadcast methods depend on the global `Universe` singleton. Calling them during early server startup before the Universe is fully initialized will result in a `NullPointerException`.
-   **Concurrent Directory Modification:** Do not allow other systems to write to the world directory while `walkFileTreeAndZip` is running. This can lead to a partial or corrupt backup. The world should be placed in a read-only or "save-disabled" state before initiating the backup process.

## Data Pipeline
The primary data flow for an archive operation is from the file system, through this utility, and back to the file system. A secondary flow communicates status to network clients.

> **Archiving Flow:**
> World Files on Disk -> `Files.walk` -> **BackupUtil.walkFileTreeAndZip** -> `ZipOutputStream` -> Zip Archive on Disk

> **Status Notification Flow:**
> Backup Orchestrator -> **BackupUtil.broadcastBackupStatus** -> Universe -> Network Packet Encoder -> Game Client UI

