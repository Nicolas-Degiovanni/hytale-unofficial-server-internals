---
description: Architectural reference for the Operation interface, the core contract for server-side interactions.
---

# Operation

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.operation
**Type:** Contract / Strategy Interface

## Definition
```java
// Signature
public interface Operation {
   void tick(
      @Nonnull Ref<EntityStore> var1,
      @Nonnull LivingEntity var2,
      boolean var3,
      float var4,
      @Nonnull InteractionType var5,
      @Nonnull InteractionContext var6,
      @Nonnull CooldownHandler var7
   );

   void simulateTick(
      @Nonnull Ref<EntityStore> var1,
      @Nonnull LivingEntity var2,
      boolean var3,
      float var4,
      @Nonnull InteractionType var5,
      @Nonnull InteractionContext var6,
      @Nonnull CooldownHandler var7
   );

   default void handle(@Nonnull Ref<EntityStore> ref, boolean firstRun, float time, @Nonnull InteractionType type, @Nonnull InteractionContext context) {
   }

   WaitForDataFrom getWaitForDataFrom();

   @Nullable
   default InteractionRules getRules() {
      return null;
   }

   default Int2ObjectMap<IntSet> getTags() {
      return Int2ObjectMaps.emptyMap();
   }

   default Operation getInnerOperation() {
      // ...
   }

   public interface NestedOperation {
      Operation inner();
   }
}
```

## Architecture & Concepts
The Operation interface is the fundamental contract for defining the logic of any server-side entity interaction. It embodies the **Strategy** design pattern, allowing the core interaction system to execute a wide variety of actions—such as attacking, using items, or blocking—without being coupled to their specific implementations.

This interface represents the "verb" in an interaction. A controlling system, such as an InteractionManager, selects a concrete Operation implementation based on player input, entity state, or environmental context. This Operation is then driven by the server's main game loop, which repeatedly calls the *tick* method to advance the interaction's state over time.

The design explicitly supports complex, multi-tick interactions. An Operation is not a single, fire-and-forget event; it is a stateful process that can evolve over its lifetime. The presence of the NestedOperation interface also suggests that Operations can be wrapped or decorated, enabling the composition of complex behaviors from simpler parts.

## Lifecycle & Ownership
- **Creation:** Concrete Operation instances are created on-demand by a higher-level interaction management system. This typically occurs in response to a client network packet indicating a player's intent to perform an action. They are **not** pre-allocated or pooled.
- **Scope:** An Operation is ephemeral and its lifetime is strictly bound to the duration of the interaction it represents. It may exist for a single server tick for instantaneous actions or persist across many ticks for charged or channeled abilities.
- **Destruction:** The instance is eligible for garbage collection as soon as the interaction concludes, is cancelled, or is superseded by another action. The managing system is responsible for dropping the reference to the Operation, allowing it to be reclaimed.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, all non-trivial implementations are expected to be **highly stateful**. They must maintain their own internal state to track progress, such as the charge level of an attack or the remaining duration of a channeled spell.
- **Thread Safety:** Implementations of Operation are **not thread-safe** and must never be considered as such. They are designed to be created, updated, and destroyed exclusively by the single thread responsible for ticking the world or region in which the interaction occurs.

**WARNING:** Accessing or modifying an Operation from any thread other than the owning game loop thread will lead to race conditions, world corruption, and server instability. All interaction logic must be confined to the synchronous server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(...) | void | Varies | The primary execution method. Called once per server tick to advance the interaction's logic. Modifies world state via the EntityStore. |
| simulateTick(...) | void | Varies | A non-authoritative tick. Likely used for speculative execution or client-side prediction validation. Should not permanently alter world state. |
| handle(...) | void | O(1) | A callback hook, typically invoked once when the Operation is first initiated to perform setup logic. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Signals to the server that this Operation cannot proceed until it receives specific data from the client. A critical component for interactive, multi-stage abilities. |
| getRules() | InteractionRules | O(1) | Returns an object defining the constraints and properties of this interaction, such as range, targeting rules, or allowed states. |
| getInnerOperation() | Operation | O(N) | For nested or decorated Operations, unwraps the layers to return the root Operation. N is the depth of nesting. |

## Integration Patterns

### Standard Usage
An Operation is always managed by a higher-level system. The client code should never create or tick an Operation directly. The typical flow involves a manager creating an instance and driving it via the game loop.

```java
// Executed within a server-side interaction management system
class InteractionManager {
    private Operation currentOperation;

    void processInteraction(InteractionIntent intent) {
        // A factory creates the appropriate Operation based on the intent
        this.currentOperation = OperationFactory.createFrom(intent);
        
        // The 'handle' method is called once on initialization
        this.currentOperation.handle(...);
    }

    void onServerTick(float deltaTime) {
        if (this.currentOperation != null) {
            // The operation is ticked every frame to advance its state
            this.currentOperation.tick(..., deltaTime, ...);

            // Logic to determine if the operation is complete...
            if (this.currentOperation.isFinished()) {
                this.currentOperation = null;
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Reuse:** Never reuse an Operation instance for a new interaction. Each interaction must have a fresh instance to ensure state is not carried over, which would cause unpredictable behavior.
- **Long-Term Storage:** Do not store references to Operation instances beyond the scope of the interaction itself. This will cause memory leaks and prevent the garbage collector from reclaiming resources.
- **Asynchronous Execution:** Do not attempt to call *tick* or any other method from a separate thread. All state mutations must occur synchronously within the main server tick.

## Data Pipeline
The Operation interface is a central processing stage in the server's interaction pipeline. It translates a high-level intent into concrete changes in the game world.

> Flow:
> Client Input Packet -> Network Protocol Layer -> **InteractionManager creates concrete Operation** -> Server Game Loop calls **tick()** -> **Operation modifies EntityStore** -> World State Change -> State Synchronization Layer -> Server Output Packet -> Client Visuals

