---
description: Architectural reference for PermissionProvider
---

# PermissionProvider

**Package:** com.hypixel.hytale.server.core.permissions.provider
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface PermissionProvider {
   @Nonnull
   String getName();

   void addUserPermissions(@Nonnull UUID var1, @Nonnull Set<String> var2);

   void removeUserPermissions(@Nonnull UUID var1, @Nonnull Set<String> var2);

   Set<String> getUserPermissions(@Nonnull UUID var1);

   void addGroupPermissions(@Nonnull String var1, @Nonnull Set<String> var2);

   void removeGroupPermissions(@Nonnull String var1, @Nonnull Set<String> var2);

   Set<String> getGroupPermissions(@Nonnull String var1);

   void addUserToGroup(@Nonnull UUID var1, @Nonnull String var2);

   void removeUserFromGroup(@Nonnull UUID var1, @Nonnull String var2);

   Set<String> getGroupsForUser(@Nonnull UUID var1);
}
```

## Architecture & Concepts

The PermissionProvider interface is a foundational contract within the server's security and authorization subsystem. It defines a standardized API for querying and manipulating permission data, abstracting the underlying storage mechanism. This is a classic example of the **Strategy Pattern**, decoupling the core permission logic from how and where permissions are actually stored (e.g., in a flat file, a relational database, or an external service).

This abstraction is critical for server modularity and extensibility. The server core interacts solely with this interface, allowing server administrators to swap out the entire permission backend by changing a configuration setting, without requiring any changes to the game logic or plugins that depend on permission checks.

The model is based on three core entities:
1.  **Users:** Identified by a unique UUID.
2.  **Groups:** Identified by a unique String name.
3.  **Permissions:** Simple String nodes (e.g., hytale.command.gamemode).

A user can have permissions directly assigned to them and can also be a member of multiple groups, inheriting all permissions from those groups.

### Lifecycle & Ownership

As an interface, PermissionProvider itself has no lifecycle. The following applies to any **concrete implementation** of this contract.

-   **Creation:** A single, specific implementation (e.g., FilePermissionProvider) is instantiated by the central PermissionService during server bootstrap. The choice of which implementation to load is determined by the server's primary configuration file.
-   **Scope:** The provider instance is a long-lived singleton, persisting for the entire duration of the server's runtime. It is held as a private field within the PermissionService.
-   **Destruction:** The provider is eligible for garbage collection only upon server shutdown when the PermissionService is destroyed. Implementations that manage external resources (like database connections) must expose a shutdown or close method to be called during this phase.

## Internal State & Concurrency

The contract does not enforce state management, but any performant implementation is expected to be highly stateful and thread-safe.

-   **State:** Concrete implementations will almost certainly maintain an in-memory cache of the entire permission graph (users, groups, and their relationships). This state is highly mutable, as permissions can be granted or revoked by administrators at any time. The provider is the source of truth for this data at runtime.
-   **Thread Safety:** **CRITICAL:** Permission checks can originate from any thread handling player input, network events, or scheduled tasks. Therefore, all implementations of PermissionProvider **must be thread-safe**. Access to internal collections must be synchronized. The use of concurrent collections (e.g., ConcurrentHashMap) or fine-grained locking (e.g., ReentrantReadWriteLock) is mandated to prevent race conditions and ensure data consistency. Writes (e.g., addUserToGroup) must be atomic and their effects must be immediately visible to all subsequent reads.

## API Surface

The API defines the fundamental Create, Read, and Delete operations for the permissions system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getName() | String | O(1) | Returns the unique name of the provider implementation (e.g., "file", "mysql"). |
| addUserPermissions(uuid, perms) | void | O(N) | Adds a set of permission nodes directly to a user. |
| removeUserPermissions(uuid, perms) | void | O(N) | Removes a set of permission nodes from a user. |
| getUserPermissions(uuid) | Set<String> | O(1) | Retrieves only the permissions assigned directly to a user. Does not include inherited permissions. |
| addGroupPermissions(group, perms) | void | O(N) | Adds a set of permission nodes to a group. |
| removeGroupPermissions(group, perms) | void | O(N) | Removes a set of permission nodes from a group. |
| getGroupPermissions(group) | Set<String> | O(1) | Retrieves the permissions assigned to a specific group. |
| addUserToGroup(uuid, group) | void | O(1) | Assigns a user to a group. |
| removeUserFromGroup(uuid, group) | void | O(1) | Removes a user from a group. |
| getGroupsForUser(uuid) | Set<String> | O(1) | Retrieves all groups a user is a member of. |

*Complexity is estimated for a typical in-memory cache implementation. N is the number of permissions in the input set.*

## Integration Patterns

### Standard Usage

Direct interaction with the PermissionProvider is discouraged. All permission-related logic should flow through the central PermissionService, which may add additional layers of caching, event dispatching, or permission resolution logic (e.g., handling wildcards or inheritance).

```java
// Correct: Interacting with the higher-level service
PermissionService permService = serverContext.getService(PermissionService.class);

// The service internally delegates to the configured PermissionProvider
if (permService.hasPermission(player.getUuid(), "hytale.world.edit")) {
    // Allow action
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Implementation Access:** Do not attempt to cast the provider to a concrete type to access implementation-specific methods. This breaks the abstraction and will fail if the provider is changed.
-   **External Caching:** Do not cache the results of provider methods (e.g., `getGroupsForUser`) in your own systems for extended periods. The underlying data can be modified at any time by server commands, and your cache will become stale. Always query the authoritative service.
-   **Bypassing the Provider:** Implementing a separate, ad-hoc permission system for a specific feature is a critical anti-pattern. It fragments the server's security model and creates confusion for administrators. All authorization checks must use the central system.

## Data Pipeline

The PermissionProvider is a critical component in the server's authorization data flow. It acts as the query engine for permission checks.

> Flow:
> Player Action (e.g., Command Execution) -> Command Handler -> PermissionService.hasPermission(uuid, node) -> **PermissionProvider.get...()** -> In-Memory Cache / Database -> Boolean Result -> Command Handler -> Action Allowed/Denied

