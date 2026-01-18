---
description: Architectural reference for PermissionsModule
---

# PermissionsModule

**Package:** com.hypixel.hytale.server.core.permissions
**Type:** Singleton

## Definition
```java
// Signature
public class PermissionsModule extends JavaPlugin {
```

## Architecture & Concepts
The PermissionsModule is the central authority for all permission and privilege enforcement within the Hytale server. It is implemented as a core JavaPlugin, ensuring it is loaded and initialized early in the server's lifecycle, making its services available to all other systems and plugins.

The architecture is designed around a pluggable provider model. The PermissionsModule itself does not store permission data; instead, it orchestrates and queries a chain of registered PermissionProvider objects. This allows the underlying permission data source to be swapped or extended, for example, to read from a database instead of a local file. By default, it is initialized with a single HytalePermissionsProvider which handles standard file-based permission storage.

A key architectural concept is the distinction between persisted and virtual permissions:
*   **Persisted Permissions:** Standard user and group permissions that are explicitly defined and stored by a PermissionProvider.
*   **Virtual Groups:** A mechanism for granting a set of implicit permissions to a group without permanently storing them. For example, the system automatically grants the *hytale.editor.builderTools* permission to any group that is also a Creative game mode group. This decouples game state from the permissions file.

When a permission check is requested, the module iterates through all registered providers in order. The first provider to return a definitive grant or deny (TRUE or FALSE) for the permission node determines the result. This chain-of-responsibility pattern allows for complex permission overrides and layering.

## Lifecycle & Ownership
- **Creation:** The PermissionsModule is instantiated by the server's internal PluginManager during the core plugin bootstrap sequence. A static singleton instance is set in the constructor for global access via the static get method.
- **Scope:** Session-scoped. The module persists for the entire lifetime of the server process. Its state, including the list of providers and virtual groups, is maintained throughout.
- **Destruction:** The module is destroyed during the server shutdown sequence when the PluginManager unloads all plugins. There is no explicit public destruction method.

## Internal State & Concurrency
- **State:** The PermissionsModule maintains a mutable state, which consists of the list of registered PermissionProvider instances and the map of virtualGroups. The actual permission data (users, groups, nodes) is considered external state managed by the individual providers.

- **Thread Safety:** The module is designed for high-throughput, concurrent reads.
    - The list of providers is stored in a CopyOnWriteArrayList. This data structure makes read operations, such as iterating providers during a hasPermission check, completely lock-free and extremely fast.
    - **WARNING:** Write operations, such as addProvider and removeProvider, are expensive as they require creating a new copy of the underlying list. These methods should only be called during the server's initialization phase (e.g., from a plugin's setup method) to avoid severe performance degradation and potential race conditions.
    - State-mutating API calls like addUserPermission are **not** internally synchronized. They delegate directly to the primary PermissionProvider. The thread safety of the underlying permission data store is the responsibility of the provider's implementation. Callers must provide their own synchronization if they intend to modify permissions from multiple threads simultaneously.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | PermissionsModule | O(1) | Accesses the global singleton instance. |
| addProvider(provider) | void | O(N) | Registers a new permission provider. Expensive; for initialization use only. |
| hasPermission(uuid, id) | boolean | O(P * G) | Checks if a user has a permission node. P is providers, G is groups. |
| addUserPermission(uuid, perms) | void | O(1) | Adds permissions to a user. Dispatches a PlayerPermissionChangeEvent. |
| removeUserPermission(uuid, perms) | void | O(1) | Removes permissions from a user. Dispatches a PlayerPermissionChangeEvent. |
| addGroupPermission(group, perms) | void | O(1) | Adds permissions to a group. Dispatches a GroupPermissionChangeEvent. |
| addUserToGroup(uuid, group) | void | O(1) | Adds a user to a group. Dispatches a PlayerGroupEvent. |
| areProvidersTampered() | boolean | O(1) | Checks if any non-standard providers are registered. |

## Integration Patterns

### Standard Usage
The primary interaction pattern is to retrieve the singleton instance and perform a permission check. This is a fundamental operation for systems like command handling and feature access control.

```java
// Check if a player has permission to use a specific command
PermissionsModule permissions = PermissionsModule.get();
UUID playerUUID = player.getUUID();

if (permissions.hasPermission(playerUUID, "hytale.command.gamemode")) {
    // Proceed with command logic
} else {
    // Deny access
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new PermissionsModule()`. The class is a managed singleton. Always use the static `PermissionsModule.get()` method to retrieve the active instance.
- **Post-Startup Provider Modification:** Do not call `addProvider` or `removeProvider` after the server has finished its startup sequence. The use of CopyOnWriteArrayList makes these operations a significant performance risk on a live server.
- **Bypassing the Module:** Avoid accessing a PermissionProvider directly. All permission checks and modifications must go through the PermissionsModule to ensure the entire provider chain and event system are respected.
- **Assuming Mutation is Thread-Safe:** Do not call methods like `addUserPermission` from multiple threads without external locking. The module does not guarantee the thread safety of the underlying data store.

## Data Pipeline
The data flow for a typical permission check follows a clear path of delegation and aggregation through the provider chain.

> Flow:
> System Call (e.g., CommandManager) -> `PermissionsModule.hasPermission(uuid, node)` -> **PermissionsModule** -> Iterates `[PermissionProvider]` -> Resolves User, Group, and Virtual Permissions -> Final Boolean Result

