---
description: Architectural reference for HytalePermissionsProvider
---

# HytalePermissionsProvider

**Package:** com.hypixel.hytale.server.core.permissions.provider
**Type:** Singleton

## Definition
```java
// Signature
public final class HytalePermissionsProvider extends BlockingDiskFile implements PermissionProvider {
```

## Architecture & Concepts
The HytalePermissionsProvider is the canonical implementation for persisting and managing all server-side user and group permissions. It serves as the single source of truth for the authorization layer, determining what actions a player can perform.

Architecturally, this class functions as a **file-backed, in-memory cache**. It maintains the entire permissions dataset in memory using high-performance maps for rapid lookups, while ensuring data durability by persisting changes to a central `permissions.json` file on disk.

The core of its design relies on its parent class, BlockingDiskFile. This inheritance provides two critical capabilities:
1.  **Atomic File I/O:** It abstracts the complexity of safely reading from and writing to the JSON file.
2.  **Concurrency Control:** It provides a built-in `ReentrantReadWriteLock`, which this class leverages to make all operations thread-safe.

This component is fundamental to server security and administration, bridging the gap between runtime permission checks and persistent server configuration.

### Lifecycle & Ownership
- **Creation:** A single instance is created by the server's core service manager during the bootstrap sequence. The constructor immediately triggers an initial, blocking load of `permissions.json` from disk into memory. If the file does not exist, an empty one is created.
- **Scope:** The object is a session-scoped singleton. It persists for the entire lifetime of the server process. Its in-memory state is the live representation of all server permissions.
- **Destruction:** The instance is dereferenced and eligible for garbage collection upon server shutdown. There are no explicit shutdown hooks within this class, as the underlying file handles are managed by the parent and the JVM.

## Internal State & Concurrency
- **State:** The internal state is highly **mutable** and is composed of three primary data structures:
    - `userPermissions`: A map of UUID to a set of specific permission strings for that user.
    - `groupPermissions`: A map of a group name to a set of permission strings for that group.
    - `userGroups`: A map linking a user's UUID to the set of groups they belong to.
    These maps serve as a complete, in-memory cache of the `permissions.json` file.

- **Thread Safety:** This class is **thread-safe**. All public methods that read or modify internal state are protected by a `ReentrantReadWriteLock` inherited from BlockingDiskFile.
    - **Read Operations** (e.g., getUserPermissions) acquire a read lock, allowing for concurrent reads from multiple threads.
    - **Write Operations** (e.g., addUserToGroup) acquire an exclusive write lock, blocking all other read and write operations until the modification is complete.

    **WARNING:** Every state modification immediately triggers a synchronous, blocking save to disk via the `syncSave` method. This guarantees data integrity but can introduce significant I/O latency. High-frequency write operations will severely impact server performance.

## API Surface
The public API conforms to the PermissionProvider interface, providing a comprehensive contract for managing users, groups, and their associated permissions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addUserPermissions(uuid, perms) | void | O(1) + I/O | Adds a set of permissions to a user. Triggers a blocking file write. |
| removeUserPermissions(uuid, perms) | void | O(1) + I/O | Removes a set of permissions from a user. Triggers a blocking file write. |
| getUserPermissions(uuid) | Set<String> | O(1) | Retrieves a user's direct permissions. Returns an unmodifiable set. |
| addGroupPermissions(group, perms) | void | O(1) + I/O | Adds a set of permissions to a group. Triggers a blocking file write. |
| getGroupPermissions(group) | Set<String> | O(1) | Retrieves a group's permissions. Returns an unmodifiable set. |
| addUserToGroup(uuid, group) | void | O(1) + I/O | Assigns a user to a group. Triggers a blocking file write. |
| removeUserFromGroup(uuid, group) | void | O(1) + I/O | Removes a user from a group. Triggers a blocking file write. |
| getGroupsForUser(uuid) | Set<String> | O(1) | Retrieves the groups a user belongs to. Returns an unmodifiable set. |

## Integration Patterns

### Standard Usage
This provider should be retrieved from the central server context or service registry. It should not be instantiated directly. It is the backend for higher-level systems like command handlers or event listeners that need to perform authorization checks.

```java
// Correctly obtain the provider from the server context
PermissionProvider permissions = serverContext.getService(PermissionProvider.class);

// Check if a player has a specific permission
UUID playerUuid = ...;
Set<String> userGroups = permissions.getGroupsForUser(playerUuid);

boolean hasPermission = userGroups.stream()
    .flatMap(group -> permissions.getGroupPermissions(group).stream())
    .anyMatch(perm -> perm.equals("hytale.command.gamemode") || perm.equals("*"));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HytalePermissionsProvider()`. Doing so will create a disconnected instance that does not represent the server's true state and may lead to file lock contention.
- **High-Frequency Writes:** Do not call modification methods (e.g., addUserToGroup) in a tight loop or on a per-tick basis. Each call blocks the calling thread and performs disk I/O, which will cause severe performance degradation. Permission changes should be batched logically by a higher-level system if possible.
- **Modifying Returned Collections:** The sets returned by getter methods are unmodifiable. Attempting to modify them will result in an `UnsupportedOperationException`.

## Data Pipeline
The component's primary function is to synchronize its in-memory state with the `permissions.json` file on disk.

> **Write Flow:**
> API Call (e.g., `addUserToGroup`) -> Acquire Write Lock -> Modify In-Memory Map -> `syncSave()` -> `write()` -> GSON Serialization -> `permissions.json`

> **Read Flow (Startup):**
> Server Boot -> `HytalePermissionsProvider` Instantiation -> `syncLoad()` -> `read()` -> `permissions.json` -> GSON Deserialization -> Populate In-Memory Maps

> **Read Flow (Runtime):**
> API Call (e.g., `getGroupsForUser`) -> Acquire Read Lock -> Access In-Memory Map -> Return Unmodifiable Set

