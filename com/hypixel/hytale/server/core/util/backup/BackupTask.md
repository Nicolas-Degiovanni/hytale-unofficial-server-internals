---
description: Architectural reference for BackupTask
---

# BackupTask

**Package:** com.hypixel.hytale.server.core.util.backup
**Type:** Transient Task Runner

## Definition
```java
// Signature
public class BackupTask {
```

## Architecture & Concepts
The BackupTask is a self-contained, asynchronous component responsible for executing the entire server world backup process. Its primary architectural goal is to perform a potentially long-running, I/O-intensive operation without blocking the main server thread, thereby preventing server stalls or freezes.

This is achieved by encapsulating the entire backup logic within a new, dedicated thread named "Backup Runner". This thread is intentionally configured as a **non-daemon thread**. This is a critical design choice ensuring that the Java Virtual Machine will not exit until a backup in progress is fully completed, safeguarding against data corruption during server shutdown sequences.

The class acts as a fire-and-forget task orchestrator. The static factory method *start* initiates the process and immediately returns a CompletableFuture, allowing the calling system to track completion or handle failures asynchronously without direct interaction with the underlying thread. The task's responsibilities include directory creation, rotational cleanup of old backups, archiving of milestone backups, and compressing the target universe directory into a timestamped zip archive.

## Lifecycle & Ownership
- **Creation:** A BackupTask instance is created exclusively through the static `start` factory method. The constructor is private to enforce this pattern and prevent misuse. The caller never holds a direct reference to the BackupTask object itself.

- **Scope:** The object's lifetime is strictly bound to the execution of the "Backup Runner" thread it spawns. It exists only to manage the state of a single backup operation.

- **Destruction:** Once the thread's `run` method completes—either through success or an exception—the thread terminates. The BackupTask instance, having no other references, becomes eligible for garbage collection. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The primary state is a single, mutable field: the `completion` CompletableFuture. This object transitions from a pending to a completed state, representing the outcome of the backup operation. All other state, such as file paths and configuration values, is passed during creation or resolved within the task's execution scope.

- **Thread Safety:** This class is **not thread-safe** and is not designed to be. Its internal state is confined to the single "Backup Runner" thread created upon instantiation. The public static `start` method is safe to call from any thread, but the instance it creates is owned and operated on exclusively by its private thread. This thread confinement model obviates the need for explicit locks or synchronization primitives.

## API Surface
The public contract is minimal, consisting of a single static method to initiate the process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| start(universeDir, backupDir) | CompletableFuture<Void> | O(N) | Initiates the asynchronous backup process. N is the size of the universe directory. Returns a future that completes when the backup is finished or fails. |

## Integration Patterns

### Standard Usage
The intended use is to trigger the backup and optionally attach non-blocking callbacks to the returned future for logging or further actions.

```java
// How a developer should normally use this
Path universe = Paths.get("./universe");
Path backups = Paths.get("./backups");

CompletableFuture<Void> backupFuture = BackupTask.start(universe, backups);

backupFuture.whenComplete((result, error) -> {
    if (error != null) {
        SERVER_LOGGER.error("Backup failed!", error);
    } else {
        SERVER_LOGGER.info("Backup completed successfully.");
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to instantiate this class using reflection. Always use the static `start` method.

- **Blocking the Server Thread:** Calling `get()` or `join()` on the returned CompletableFuture from the main server thread is a severe anti-pattern. This will block the thread and freeze the server, defeating the entire purpose of the asynchronous design.

```java
// ANTI-PATTERN: This will freeze the server until the backup is complete.
CompletableFuture<Void> backupFuture = BackupTask.start(universe, backups);
backupFuture.get(); // DO NOT DO THIS on a critical thread.
```

## Data Pipeline
The BackupTask orchestrates a data flow primarily between the server's filesystem and its internal logic, driven by configuration values.

> Flow:
> `BackupTask.start` Call -> Read `Options.BACKUP_MAX_COUNT` -> Filesystem Scan (Existing Backups) -> Filesystem Delete/Move (Cleanup & Archive) -> Filesystem Read (Universe Directory) -> In-Memory Compression -> Filesystem Write (Temporary .zip.tmp) -> Filesystem Atomic Move (Final .zip) -> `CompletableFuture` Completion

