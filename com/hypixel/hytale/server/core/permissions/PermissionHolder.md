---
description: Architectural reference for the PermissionHolder interface, the core contract for permission-aware entities in the server.
---

# PermissionHolder

**Package:** com.hypixel.hytale.server.core.permissions
**Type:** Interface

## Definition
```java
// Signature
public interface PermissionHolder {
```

## Architecture & Concepts
The PermissionHolder interface is a fundamental security and gameplay abstraction within the server architecture. It establishes a standard contract for any object that can possess permissions, such as players, groups, or server roles.

This interface decouples the act of permission checking from the underlying implementation. Systems that need to authorize an action do not need to know the concrete type of the entity they are querying. They simply interact with the PermissionHolder contract, asking "Does this entity have permission for node X?". This design is critical for creating a flexible and extensible permissions system where new types of permission-aware entities can be introduced without modifying existing authorization logic.

It serves as the public-facing view of an entity's capabilities, abstracting away the complexities of how those permissions are derived, whether from a database, a configuration file, inheritance from parent groups, or temporary grants.

### Lifecycle & Ownership
As an interface, PermissionHolder does not have its own lifecycle. Instead, its lifecycle is intrinsically tied to the object that implements it.

- **Creation:** An object that implements PermissionHolder is created according to its own specific lifecycle. For example, a Player object, which implements this interface, is created when a user connects to the server.
- **Scope:** The scope of a PermissionHolder is identical to the scope of its implementing object. It exists as long as the parent object (e.g., Player) exists.
- **Destruction:** The PermissionHolder contract is no longer valid once the implementing object is garbage collected or explicitly destroyed (e.g., on player disconnect).

## Internal State & Concurrency
- **State:** This interface is stateless by definition. All state, such as the list of granted permission nodes, is managed by the concrete implementation. Implementations are expected to be mutable, as permissions can be granted or revoked at runtime.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. Permission checks can be invoked from numerous contexts, including the main server thread, network I/O threads processing commands, or asynchronous task threads. Failure to ensure thread safety in the implementing class will lead to severe race conditions and unpredictable server behavior. Implementers should use concurrent collections or appropriate locking mechanisms to protect their internal state.

## API Surface
The public contract is minimal, focusing exclusively on querying permission status.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasPermission(String permission) | boolean | O(log N) to O(N) | Checks if the holder has the specified permission node. The complexity depends on the implementation's data structure. Returns false if the permission is not explicitly defined. |
| hasPermission(String permission, boolean defaultValue) | boolean | O(log N) to O(N) | Checks for the permission node, returning the provided default value if the permission is not explicitly set for the holder. This is the preferred method for checking optional or non-critical permissions. |

## Integration Patterns

### Standard Usage
Code should always depend on the interface, not a concrete implementation. Retrieve the PermissionHolder from the relevant entity (like a Player or CommandSource) and perform the check.

```java
// Correctly check if a player can use a world modification command
Player player = server.getPlayer("Notch");
PermissionHolder permissions = (PermissionHolder) player;

if (permissions.hasPermission("hytale.world.edit")) {
    // Proceed with action
} else {
    player.sendMessage("You do not have permission to do that.");
}
```

### Anti-Patterns (Do NOT do this)
- **Implementation Casting:** Never cast a PermissionHolder to a specific type to access implementation details. This violates the principle of the interface and creates brittle code.
- **External Caching:** Do not cache the result of a permission check in an external system. The implementing class is responsible for its own caching strategy. External caching can lead to stale authorization data if a player's permissions are changed at runtime.
- **Assuming Default Implementation:** Do not assume that the underlying implementation is a simple list or map. It may involve complex inheritance lookups from multiple parent groups.

## Data Pipeline
PermissionHolder acts as a query endpoint in a control flow rather than a step in a data pipeline. It is typically the final gate in an authorization workflow.

> Flow:
> Network Command Packet -> Command Dispatcher -> Get CommandSource (e.g., Player) -> **PermissionHolder.hasPermission("command.name")** -> Boolean Result -> Allow/Deny Command Execution

