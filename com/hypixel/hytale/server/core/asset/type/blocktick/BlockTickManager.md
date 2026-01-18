---
description: Architectural reference for BlockTickManager
---

# BlockTickManager

**Package:** com.hypixel.hytale.server.core.asset.type.blocktick
**Type:** Static Service Locator

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public final class BlockTickManager {
```

## Architecture & Concepts

The BlockTickManager is a static, global service locator designed to hold a single, server-wide instance of an **IBlockTickProvider**. Its primary architectural role is to decouple game systems that require block ticking services from the concrete implementation of that service. This allows the underlying block tick mechanism to be swapped out without affecting consumer code.

This class implements a global singleton pattern for the provider it holds. It is a central, static registry that any part of the server can access without needing a reference to a specific context or world instance.

**WARNING:** This class is marked as deprecated and is scheduled for complete removal. Its use of a global static variable is a legacy pattern that is being phased out in favor of context-aware dependency injection systems. New code **MUST NOT** integrate with BlockTickManager. Existing systems should be migrated away from it.

## Lifecycle & Ownership

-   **Creation:** The BlockTickManager class is never instantiated; its constructor is private. Its static state is initialized by the JVM during class loading. The internal **BLOCK_TICK_PROVIDER** reference is initialized to a default, non-functional **IBlockTickProvider.NONE** instance at this time. The *actual* provider is injected later in the server bootstrap process by calling **setBlockTickProvider**.
-   **Scope:** The state held by this class is global and static. It persists for the entire lifetime of the server application.
-   **Destruction:** The singleton provider instance is held until the JVM shuts down. There is no explicit cleanup or teardown method.

## Internal State & Concurrency

-   **State:** The class manages a single, mutable, static field: **BLOCK_TICK_PROVIDER**. This state is shared across all threads in the server process.
-   **Thread Safety:** This class is fully thread-safe. The internal state is managed by an **AtomicReference**, which guarantees that all reads and writes to the provider reference are atomic. This prevents race conditions when one thread is setting the provider while another is attempting to retrieve it. Any thread may safely call its methods at any time.

## API Surface

The public API consists entirely of static methods for managing the global **IBlockTickProvider** instance.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlockTickProvider(provider) | void | O(1) | Atomically sets the global provider instance. |
| getBlockTickProvider() | IBlockTickProvider | O(1) | Atomically retrieves the currently configured provider. |
| hasBlockTickProvider() | boolean | O(1) | Returns true if the provider is not the default **IBlockTickProvider.NONE**. |

## Integration Patterns

### Standard Usage

The intended pattern is for a high-level system, such as the main server entry point, to configure the provider once during startup. Other game systems then retrieve and use this provider throughout the server's lifetime.

```java
// During server initialization phase
// A concrete implementation is created and registered globally.
IBlockTickProvider worldTickProvider = new ConcreteWorldTickProvider(world);
BlockTickManager.setBlockTickProvider(worldTickProvider);

// Elsewhere, in a system that needs to schedule a block update
if (BlockTickManager.hasBlockTickProvider()) {
    IBlockTickProvider provider = BlockTickManager.getBlockTickProvider();
    provider.scheduleTick(position, block, delay);
}
```

### Anti-Patterns (Do NOT do this)

-   **Re-initialization:** Do not call **setBlockTickProvider** more than once. The system is designed for a single, definitive provider to be set during the server bootstrap sequence. Overwriting it at runtime can lead to unpredictable behavior and lost ticks.
-   **Usage in New Code:** Do not use this manager in any new feature development. Its deprecated status means it will be removed, and its global nature is contrary to modern, context-specific architecture.
-   **Assuming a Provider Exists:** Do not call **getBlockTickProvider** and use it without first checking **hasBlockTickProvider** or being certain that initialization has completed. Early in the server lifecycle, the provider will be the non-functional **NONE** instance.

## Data Pipeline

BlockTickManager does not process data itself. It acts as a control-flow component that directs dependencies. Its role is to hold and provide a service, not to transform data.

> **Dependency Flow:**
> Server Bootstrap -> Instantiates `ConcreteBlockTickProvider` -> **BlockTickManager.setBlockTickProvider()** -> Game Logic Systems -> **BlockTickManager.getBlockTickProvider()** -> `ConcreteBlockTickProvider.scheduleTick()`

