---
description: Architectural reference for BlockStateRegistry
---

# BlockStateRegistry

**Package:** com.hypixel.hytale.server.core.universe.world.meta
**Type:** Singleton

## Definition
```java
// Signature
public class BlockStateRegistry extends Registry<BlockStateRegistration> {
```

## Architecture & Concepts
The BlockStateRegistry is a critical component of the server's world initialization system. It serves as the authoritative, lifecycle-aware facade for defining all possible block states within the game universe. Its primary architectural role is not to store the block state data itself, but to act as a gatekeeper, strictly enforcing *when* block states can be registered.

This class operates on a fundamental principle of engine initialization: a clear separation between a dynamic "registration phase" and an immutable "runtime phase". This is enforced via a `precondition` BooleanSupplier provided during its construction. Once this precondition fails (e.g., the server has finished loading mods and is about to start the game loop), the registry is effectively "frozen," and any further attempts to register new block states will fail with an exception.

Internally, all registration operations are delegated to the central **BlockStateModule**. The BlockStateRegistry's responsibility is therefore policy enforcement, while the BlockStateModule handles the mechanism of storing and managing the state definitions. This separation of concerns ensures that engine-level lifecycle rules are respected without coupling them directly to the underlying data storage.

## Lifecycle & Ownership
- **Creation:** A single instance is instantiated by the core server bootstrap sequence, likely within a world or universe initialization module. It is created very early in the server startup process, before any mods or game content are loaded.
- **Scope:** The BlockStateRegistry is a session-scoped singleton. It persists for the entire lifetime of a running server instance.
- **Destruction:** The object is eligible for garbage collection upon server shutdown. No explicit cleanup methods are exposed or required.

## Internal State & Concurrency
- **State:** The class inherits a collection of BlockStateRegistration objects from its parent Registry. This state is mutable exclusively during the server's initial loading phase. Once the registration precondition becomes false, the internal collection should be considered immutable for the remainder of the server's lifecycle.
- **Thread Safety:** **This class is not thread-safe for write operations.** All registration calls must be performed from the main server thread during the designated initialization phase. Concurrent registration attempts will lead to unpredictable behavior and potential race conditions within the underlying BlockStateModule. Read operations after the registry is frozen are safe.

## API Surface
The public API is intentionally minimal, focusing solely on the act of registration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerBlockState(Class, String, Codec) | BlockStateRegistration | O(1) | Registers a block state with its associated class, key, and network codec. Throws an IllegalStateException if the registration precondition is not met. |
| registerBlockState(Class, String, Codec, Class, Codec) | BlockStateRegistration | O(1) | Registers a block state that includes additional, serializable StateData. Throws an IllegalStateException if the registration precondition is not met. |

## Integration Patterns

### Standard Usage
Registration should occur within a dedicated initialization block, where access to the server's central registry context is provided. The developer defines the block state's class, a unique string identifier, and the necessary codecs for serialization.

```java
// Within a mod's initialization entry point
BlockStateRegistry registry = serverContext.getService(BlockStateRegistry.class);

// Register a simple custom block state
registry.registerBlockState(
    CustomStoneBlock.class,
    "my_mod:custom_stone",
    CUSTOM_STONE_CODEC
);

// Register a complex block state with associated data
registry.registerBlockState(
    CustomFurnaceBlock.class,
    "my_mod:custom_furnace",
    CUSTOM_FURNACE_CODEC,
    CustomFurnaceData.class,
    CUSTOM_FURNACE_DATA_CODEC
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockStateRegistry()`. The instance is managed by the engine's core services and must be retrieved from the appropriate context or service locator. Direct instantiation will result in a disconnected registry that has no effect on the game.
- **Late Registration:** Do not attempt to register block states after the server has finished its primary loading phase (e.g., in response to a player command or game event). This will violate the registry's lifecycle precondition and throw a runtime exception, halting the server. All block states must be known at startup.

## Data Pipeline
The BlockStateRegistry is a definitional component that exists at the very beginning of the data lifecycle for blocks. It populates the systems that other parts of the engine later consume.

> **Registration Flow:**
> Mod or Core Game Code -> **BlockStateRegistry.registerBlockState()** -> BlockStateModule -> Global Block State Table -> World Generation & Network Serialization

