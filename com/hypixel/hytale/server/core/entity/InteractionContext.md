---
description: Architectural reference for InteractionContext
---

# InteractionContext

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Transient

## Definition
```java
// Signature
public class InteractionContext {
```

## Architecture & Concepts

The InteractionContext is a foundational, stateful object within the server's entity interaction system. It functions as a "parameter object" that encapsulates all the necessary state for a single, ongoing interaction, such as a player attacking, using an item, or blocking.

Its primary role is to act as a mobile state container that is passed through the various stages of an interaction's execution. It decouples the interaction logic (defined in `InteractionOperation`s) from the raw entity state, providing a consistent and extensible API for manipulating the world in response to an action.

Key architectural components include:

*   **Entity References:** It holds references to the `owningEntity` (the initiator of the action) and potentially a different `runningForEntity` in cases of proxying.
*   **Item State:** It tracks the `heldItem` that initiated the interaction, preserving the `originalItemType` even if the item stack is modified or consumed during the process.
*   **State Machine Link:** It is tightly coupled with an `InteractionChain` and an `InteractionEntry`, which represent the state machine and the current state of the interaction, respectively. The context provides the data, while the chain provides the logic flow.
*   **DynamicMetaStore:** The `metaStore` is a critical feature, implementing a property bag pattern. This allows different parts of the interaction system to pass arbitrary, contextual data (e.g., `TARGET_ENTITY`, `HIT_LOCATION`) without requiring a rigid, monolithic data structure. This makes the system highly extensible.
*   **Forking Mechanism:** The `fork` method is a powerful concept that allows a single interaction to branch into multiple, independent sub-interactions. For example, a single sword swing could fork into separate damage and effect chains for each entity it hits.

This class is the central nervous system for any complex entity action, carrying both the initial state and the evolving results of the action as it progresses.

### Lifecycle & Ownership

The lifecycle of an InteractionContext is ephemeral and strictly tied to the duration of a single `InteractionChain`.

*   **Creation:** An InteractionContext is never instantiated directly via its constructor. It is created exclusively through static factory methods like `forInteraction` or `forProxyEntity`. The `InteractionManager` is the primary client, creating a context in response to network input or server-side events that trigger an entity action.
*   **Scope:** The object's lifetime spans the execution of one complete interaction. It is initialized by the `InteractionManager`, passed to an `InteractionChain`, and then used by sequential `InteractionOperation`s within that chain. Once the chain terminates (either by completing, being cancelled, or being replaced), the context's purpose is fulfilled.
*   **Destruction:** The object is managed by the Java garbage collector. Ownership is held by the `InteractionManager` and the active `InteractionChain`. The package-private `initEntry` and `deinitEntry` methods signal the binding and unbinding of the context to a specific part of the state machine, effectively managing its active scope. When the chain is destroyed and the manager releases its reference, the context becomes eligible for garbage collection.

## Internal State & Concurrency

*   **State:** The InteractionContext is highly **mutable**. Its fields, particularly the `heldItem`, `chain`, `entry`, and the internal map of its `metaStore`, are expected to be modified throughout the interaction lifecycle. It is designed to be a transient state carrier, not an immutable data record.

*   **Thread Safety:** This class is **not thread-safe**. It contains no internal locking or synchronization mechanisms. All access and modification must be performed on the primary server thread responsible for ticking the associated entity. Unsynchronized access from other threads will lead to race conditions, state corruption, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fork(...) | InteractionChain | O(N) | Creates a new, branching `InteractionChain` from the current context. N is the size of the copied chain data. Throws `IllegalArgumentException` if forking with the same context. |
| duplicate() | InteractionContext | O(M) | Creates a shallow copy of the context, primarily for creating new, independent contexts with similar starting data. M is the size of the `metaStore`. |
| execute(RootInteraction) | void | O(1) | Pushes a new root interaction onto the current chain, effectively changing the state machine's primary logic path. |
| jump(Label) | void | O(1) | Performs a direct jump to a different operation within the current interaction sequence, akin to a GOTO statement. |
| getEntity() | Ref<EntityStore> | O(1) | Returns a reference to the entity for which the interaction is currently running. |
| getMetaStore() | DynamicMetaStore | O(1) | Provides access to the dynamic property bag for storing and retrieving arbitrary interaction data. |
| getSnapshot(...) | EntitySnapshot | O(1) | Retrieves a historical snapshot of an entity's state, typically used for server-side lag compensation and hit validation. |
| setHeldItem(ItemStack) | void | O(1) | Updates the `ItemStack` associated with the interaction. |

## Integration Patterns

### Standard Usage

The InteractionContext is created by the `InteractionManager` at the beginning of an action. It is then used to initialize and drive an `InteractionChain`. Operations within the chain read from and write to the context's `metaStore` to communicate and pass state.

```java
// In a system like InteractionManager
// 1. Create the context based on the entity and action type.
InteractionContext context = InteractionContext.forInteraction(
    this,
    entityRef,
    InteractionType.Primary,
    componentAccessor
);

// 2. Use the context to find and start a new InteractionChain.
String interactionId = context.getRootInteractionId(InteractionType.Primary);
RootInteraction root = Interactions.get(interactionId);
InteractionChain chain = new InteractionChain(context, root);

// 3. The chain now owns and uses the context for its execution.
activeChains.add(chain);
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** The constructors are private. Never attempt to create an `InteractionContext` using `new`. Always use the provided static factory methods (`forInteraction`, `withoutEntity`, etc.) to ensure correct initialization.
*   **State Reuse:** Do not reuse an InteractionContext instance across multiple, distinct interactions. Its state is specific to the `InteractionChain` it was created for. Reusing it will cause state leakage and unpredictable behavior.
*   **Asynchronous Modification:** Never modify an InteractionContext from a separate thread. All operations must be synchronized with the main server game loop that owns the entity.
*   **External State Management:** Avoid storing state related to an ongoing interaction outside of the context's `metaStore`. The context is the single source of truth for the duration of the action.

## Data Pipeline

The InteractionContext is a central hub in the data flow of an entity interaction, transforming high-level intent into concrete game state changes.

> Flow:
> Player Input Packet -> `InteractionManager` -> **`InteractionContext` (Creation & Population)** -> `InteractionChain` (Execution) -> `InteractionOperation` (Reads from/Writes to Context) -> `CommandBuffer` (Game State Mutation) -> Network Message (Client Update)

