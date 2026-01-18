---
description: Architectural reference for SplitVelocity
---

# SplitVelocity

**Package:** com.hypixel.hytale.server.core.modules.splitvelocity
**Type:** Singleton Plugin

## Definition
```java
// Signature
public class SplitVelocity extends JavaPlugin {
```

## Architecture & Concepts
The SplitVelocity class is a core server plugin that functions as a feature-flag controller for a client-side physics optimization. Its primary responsibility is to detect if a connecting client supports the `SplitVelocity` feature and to broadcast this capability to other server systems via a global static flag.

This component acts as a bridge between the client feature negotiation process and the server's entity physics simulation. During the server's setup phase, this plugin registers its interest in the `SplitVelocity` client feature. When a client that supports this feature connects, the plugin's `setup` method is invoked, which in turn modifies the global state to disable legacy velocity calculations.

This design employs a simple but rigid pattern: a plugin's lifecycle is directly tied to a global boolean flag. This creates a strong coupling between the plugin system and any system that reads this flag, such as the physics engine. While effective for a binary feature toggle, this approach lacks scalability and introduces concurrency risks due to its reliance on a non-synchronized static variable.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's `PluginManager` during the initial server bootstrap process. It is discovered and loaded as a mandatory core plugin.
- **Scope:** The SplitVelocity instance persists for the entire lifetime of the server process. Its internal state change, however, is triggered by its lifecycle hooks.
- **Destruction:** The object is marked for garbage collection only upon server shutdown when the `PluginManager` is dismantled. The `shutdown` method is invoked during this process to reset the global state to its default.

## Internal State & Concurrency
- **State:** The class itself contains no instance-level state. Its effective state is stored in the `public static boolean SHOULD_MODIFY_VELOCITY` field. This is a mutable, global flag, initialized to `true`.
- **Thread Safety:** **This class is not thread-safe.** The static `SHOULD_MODIFY_VELOCITY` flag is accessed without any synchronization primitives like `volatile` or locks. While the `setup` and `shutdown` methods that write to this flag are expected to be called from a single main server thread, the flag can be read by any number of threads simultaneously (e.g., physics simulation threads). This creates a significant risk of data races and memory visibility issues, where a thread may read a stale value of the flag.

**WARNING:** Systems reading the `SHOULD_MODIFY_VELOCITY` flag from concurrent threads must be aware of the potential for stale reads. The lack of a memory barrier (e.g., `volatile`) means changes may not be immediately visible across all CPU cores.

## API Surface
The public contract of this class is not defined by callable methods but by its lifecycle hooks and the global flag it manipulates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setup() | protected void | O(1) | Framework-invoked method. Registers the client feature and disables the legacy velocity path by setting SHOULD_MODIFY_VELOCITY to false. |
| shutdown() | protected void | O(1) | Framework-invoked method. Re-enables the legacy velocity path by setting SHOULD_MODIFY_VELOCITY to true, restoring the default server behavior. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. Other systems integrate with it implicitly by reading its public static flag to toggle their own internal logic.

```java
// Example from a hypothetical physics system
// This code reads the global state managed by SplitVelocity.

public void updateEntityVelocity(Entity entity) {
    if (SplitVelocity.SHOULD_MODIFY_VELOCITY) {
        // Path for older clients: Apply legacy velocity modifications.
        applyLegacyVelocityLogic(entity);
    } else {
        // Path for modern clients: Rely on the new split velocity data.
        applyModernVelocityLogic(entity);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new SplitVelocity()`. The plugin's lifecycle is exclusively managed by the server's `PluginManager`.
- **External State Mutation:** Do not write to `SplitVelocity.SHOULD_MODIFY_VELOCITY` from any other part of the codebase. Modifying this flag externally will break the feature negotiation contract and lead to unpredictable physics behavior. Its state must only be controlled by this plugin's `setup` and `shutdown` methods.

## Data Pipeline
SplitVelocity does not process a continuous flow of data. Instead, it responds to a one-time setup event and alters a global control flag. The "flow" is one of control, not data.

> Control Flow:
> Server Bootstrap → PluginManager loads **SplitVelocity** → Client Connection Handshake → Client advertises `ClientFeature.SplitVelocity` support → Plugin Framework invokes **SplitVelocity.setup()** → Global flag `SHOULD_MODIFY_VELOCITY` is set to `false` → Physics Engine reads flag on next tick and alters its simulation logic.

