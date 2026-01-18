---
description: Architectural reference for PrioritisedProviderView
---

# PrioritisedProviderView

**Package:** com.hypixel.hytale.server.npc.blackboard.view
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class PrioritisedProviderView<T, ViewType extends IBlackboardView<ViewType>> implements IBlackboardView<ViewType> {
```

## Architecture & Concepts

The PrioritisedProviderView is an abstract foundational component within the server-side NPC AI framework. It serves as a generic, priority-ordered registry for "providers". In the context of Hytale's AI, a provider is typically a source of data, a behavior, or a decision-making strategy.

This class is a direct implementation of the **Strategy Pattern**, where multiple algorithms (providers) for a specific task can be registered and the highest-priority one is selected at runtime. It forms a core building block for an AI **Blackboard** system, allowing different parts of the AI to contribute potential actions or targets into a shared view, which can then be queried to make a final decision.

The core architectural function is to decouple the AI's decision-making logic from the specific sources of information. For example, a concrete subclass like a `TargetProviderView` could receive providers from the NPC's core aggro system, temporary quest objectives, and player-initiated taunts. By assigning a priority to each provider, the AI can deterministically select the most important target without the core logic needing to know about quests or taunts specifically.

Priority is defined by an integer, where a **lower numerical value indicates a higher priority**.

## Lifecycle & Ownership

- **Creation:** As an abstract class, PrioritisedProviderView is never instantiated directly. Concrete subclasses (e.g., `AttackBehaviorProviderView`, `MovementTargetProviderView`) are instantiated and managed by an NPC's `Blackboard` or equivalent top-level AI component. This typically occurs when an NPC entity is spawned and its AI controller is initialized.
- **Scope:** The lifetime of a PrioritisedProviderView instance is tightly coupled to its owning NPC entity. It persists as long as the NPC is active in the world.
- **Destruction:** The object is eligible for garbage collection when the parent NPC entity is despawned and its AI components are de-referenced. There is no explicit `destroy` or `cleanup` method defined in this base class.

## Internal State & Concurrency

- **State:** The primary state is the `providers` list, which is a mutable collection of `PrioritisedProvider` objects. This list's size and order change whenever `registerProvider` is called. It is effectively a stateful registry.

- **Thread Safety:** This class is **not thread-safe**. The internal `providers` list is an `ObjectArrayList`, which is not synchronized. Concurrent calls to `registerProvider` from multiple threads will lead to race conditions, `ConcurrentModificationException`, or a corrupted, unsorted list. All interactions with instances of this class and its subclasses must be confined to the primary server thread that ticks the NPC's AI.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LOWEST_PRIORITY | static int | O(1) | A constant representing the lowest possible priority (`Integer.MAX_VALUE`), useful for default or fallback providers. |
| registerProvider(int, T) | void | O(N log N) | Registers a provider instance with a given priority. **Warning:** This operation re-sorts the entire internal list, making it computationally expensive. It should not be called in performance-critical loops. |

## Integration Patterns

### Standard Usage

The intended pattern is to subclass PrioritisedProviderView to create a specific type of view. Other AI systems then register their provider implementations during an initialization phase. The AI's core update loop would then query the concrete view to retrieve the highest-priority provider.

```java
// Example of a concrete subclass and its usage
// Assumes the existence of an ITargetProvider interface

// 1. A concrete view is defined for providing targets
public class TargetProviderView extends PrioritisedProviderView<ITargetProvider, TargetProviderView> {
    public Optional<Entity> getHighestPriorityTarget() {
        // Logic to iterate providers and find the first valid target
        for (PrioritisedProvider<ITargetProvider> p : providers) {
            Optional<Entity> target = p.getProvider().findTarget();
            if (target.isPresent()) {
                return target;
            }
        }
        return Optional.empty();
    }
}

// 2. An AI system registers a provider during initialization
public void initializeAggroSystem(NPC npc) {
    TargetProviderView targetView = npc.getBlackboard().getView(TargetProviderView.class);
    
    // Register a provider for finding nearby hostile players with a default priority
    ITargetProvider aggroProvider = new HostilePlayerProvider(npc);
    targetView.registerProvider(100, aggroProvider);
    
    // Register a high-priority provider for special quest targets
    ITargetProvider questProvider = new QuestTargetProvider(npc);
    targetView.registerProvider(10, questProvider);
}
```

### Anti-Patterns (Do NOT do this)

- **Frequent Registration:** Do not call `registerProvider` on every game tick or in any high-frequency loop. The O(N log N) complexity of the re-sort will cause significant performance degradation. Providers should be registered once during initialization or when a major state change occurs.
- **Concurrent Modification:** Do not access or modify a view instance from asynchronous tasks or different threads without external locking. The internal state is not synchronized.
- **Stateful Providers:** While not forbidden, providers that contain mutable state can introduce complexity. The ideal provider is a stateless algorithm that operates on the current world state when its methods are invoked.

## Data Pipeline

This class does not process a stream of data. Instead, it acts as a collection point that aggregates and sorts potential data sources or behaviors. Its role is to prepare a prioritized list that other systems will query later.

> **Setup Flow:**
> AI System Initialization -> Calls **registerProvider(priority, provider)** -> PrioritisedProviderView adds provider to internal list -> `Collections.sort` is invoked -> Internal list is now sorted by priority.
>
> **Runtime Query Flow:**
> NPC AI Tick -> Concrete Subclass (e.g., `getHighestPriorityTarget`) -> Iterates internal sorted list -> Returns result from first valid provider.

