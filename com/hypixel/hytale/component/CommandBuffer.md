---
description: Architectural reference for CommandBuffer
---

# CommandBuffer

**Package:** com.hypixel.hytale.component
**Type:** Transient

## Definition
```java
// Signature
public class CommandBuffer<ECS_TYPE> implements ComponentAccessor<ECS_TYPE> {
```

## Architecture & Concepts
The CommandBuffer is the cornerstone of the engine's Entity-Component-System (ECS) concurrency model. It implements a deferred execution pattern, acting as a transactional buffer for mutations against the central ECS **Store**.

Instead of applying changes to the ECS world directly—an operation that would require expensive locking in a multithreaded environment—systems record their intended changes (e.g., creating an entity, adding a component) into a CommandBuffer. These buffers are thread-local and isolated.

At a designated synchronization point in the game loop, typically after all parallel systems have finished their updates, the engine consumes these buffers, replaying the recorded commands in a deterministic, serial manner. This architecture enables massive parallelism for game logic (systems) while guaranteeing that the world state remains consistent without the use of locks during system execution.

The CommandBuffer is not a data structure to be stored or passed around freely; it is a short-lived tool for batching state changes.

### Lifecycle & Ownership
- **Creation:** CommandBuffers are managed by an internal object pool within the ECS **Store**. They are never instantiated directly using the constructor. A system acquires a buffer by calling a method like `store.takeCommandBuffer()`. This is a critical performance feature to eliminate garbage collection overhead during the game loop.
- **Scope:** The scope of a CommandBuffer is typically bound to the execution of a single system update or a discrete logical transaction. It is acquired at the beginning of a task and should be considered invalid after being consumed.
- **Destruction:** The object is not destroyed. The `consume` method finalizes the transaction by applying the queued operations to the Store and then returns the CommandBuffer instance to the internal object pool for reuse. Failure to consume a buffer will result in a resource leak and lost ECS mutations.

## Internal State & Concurrency
- **State:** The CommandBuffer is a stateful, mutable object. Its primary internal state is the `queue`, a Deque of `Consumer` functions, which represents the ordered list of mutations to be performed.
- **Thread Safety:** **CRITICAL:** The CommandBuffer is **not thread-safe** and is strictly thread-affine. It captures the thread on which it was acquired and will throw an assertion error if accessed from any other thread. This is a core design constraint to ensure that command queues are not corrupted. Each parallel task or system *must* operate on its own unique CommandBuffer instance.

## API Surface
The public API provides the standard set of ECS mutation operations. All of these methods are non-blocking and have O(1) complexity, as they only enqueue an operation for later execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addEntity(holder, reason) | Ref | O(1) | Enqueues the creation of a new entity. Returns a temporary Ref. |
| removeEntity(ref, reason) | void | O(1) | Enqueues the removal of an existing entity. |
| addComponent(ref, type) | Component | O(1) | Enqueues the addition of a new component to an entity. |
| removeComponent(ref, type) | void | O(1) | Enqueues the removal of a component from an entity. |
| consume() | void | O(N) | Executes all N enqueued commands against the Store and returns the buffer to the pool. |
| fork() | CommandBuffer | O(1) | Acquires a new child buffer from the Store for nested or parallel sub-tasks. |

## Integration Patterns

### Standard Usage
A system acquires a CommandBuffer, performs its logic by queueing mutations, and the engine orchestrator later consumes the buffer at a synchronization barrier.

```java
// Within a system's update method
CommandBuffer<MyEcsType> cmd = store.takeCommandBuffer();

// Queue operations based on game logic
for (Ref<MyEcsType> entityRef : query.getEntities()) {
    if (shouldExplode(entityRef)) {
        cmd.addComponent(entityRef, Exploding.class);
        cmd.removeComponent(entityRef, Health.class);
    }
}

// The engine will call consume() at the end of the stage.
// The system itself does NOT call consume().
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** `new CommandBuffer(store)` is forbidden. This bypasses the object pool, leading to severe performance degradation from GC pressure and breaking the intended lifecycle.
- **Sharing Across Threads:** Passing a CommandBuffer instance to another thread will cause race conditions and trigger assertions. Each thread of execution requires its own buffer from the Store.
- **Leaking Buffers:** Failing to have a buffer be consumed (e.g., by holding a reference to it past the current game tick) is a resource leak. The buffer will never be returned to the pool.
- **Interleaved Reads:** Reading from the Store after queueing a change in a CommandBuffer will not reflect that change. The world state is only modified after the buffer is consumed.

## Data Pipeline
The CommandBuffer acts as a temporary staging area for commands. Data does not flow *through* it; rather, commands are accumulated *in* it before being executed against the authoritative Store.

> Flow:
> System Logic -> **CommandBuffer**.add/remove() -> Command added to internal Deque -> Engine Sync Point -> **CommandBuffer**.consume() -> Direct mutation of ECS **Store**

