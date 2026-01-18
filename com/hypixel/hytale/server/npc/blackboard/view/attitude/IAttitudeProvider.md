---
description: Architectural reference for IAttitudeProvider
---

# IAttitudeProvider

**Package:** com.hypixel.hytale.server.npc.blackboard.view.attitude
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface IAttitudeProvider {
```

## Architecture & Concepts
The IAttitudeProvider interface defines a formal contract for determining the disposition of one game entity towards another. It is a core component of the server-side NPC Artificial Intelligence system, operating within the Blackboard architectural pattern.

This interface embodies the Strategy pattern, allowing for the decoupling of an NPC's decision-making logic from the specific rules that govern its social and factional relationships. Different implementations of IAttitudeProvider can be swapped to produce vastly different behaviors (e.g., a hostile bandit versus a neutral merchant) without altering the core AI state machine or behavior tree.

Its primary function is to answer the question: "Given my current state and role, what is my attitude towards this other entity?" The resulting Attitude object (e.g., HOSTILE, FRIENDLY, NEUTRAL) is then written to the NPC's Blackboard, making it available to other AI systems that will select appropriate actions.

## Lifecycle & Ownership
As an interface, IAttitudeProvider itself has no lifecycle. The lifecycle described here pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated and assigned when an NPC's Role is defined or loaded from asset files. They are part of the static definition of an NPC's behavior, not created per-instance of an NPC.
- **Scope:** An IAttitudeProvider implementation is typically stateless and designed to be shared across all NPC instances that share the same Role. Its lifetime is tied to the server's asset registry for NPC Roles.
- **Destruction:** Implementations are eligible for garbage collection when their associated Role definitions are unloaded from the server, for instance, during a hot-reload of game assets or a server shutdown.

## Internal State & Concurrency
- **State:** Implementations of this interface are expected to be **stateless**. All necessary context for determining an attitude is provided via the arguments to the getAttitude method. Storing mutable state within an implementation is a severe anti-pattern that can lead to unpredictable AI behavior across different NPCs who share the same provider instance.

- **Thread Safety:** Implementations must be thread-safe. The server's AI engine may process NPC behaviors in parallel across multiple threads. The arguments, such as Ref and ComponentAccessor, are designed to provide safe, read-only access to the world state. Any logic within an implementation must not cause side effects or rely on non-thread-safe external state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAttitude(sourceEntity, sourceRole, targetEntity, accessor) | Attitude | Implementation-dependent | Calculates the attitude of the source entity towards the target entity. This is the central contract of the interface. |
| OVERRIDE_PRIORITY | int | O(1) | A static constant used to resolve conflicts when multiple providers could apply. Higher values take precedence. |

## Integration Patterns

### Standard Usage
The provider is invoked by higher-level AI systems, such as a Blackboard or a behavior tree node, to populate the NPC's understanding of its environment. The system retrieves the appropriate provider for the NPC's Role and uses it to evaluate potential targets.

```java
// Simplified example from within an AI decision-making system

// Get the provider associated with the NPC's current role
IAttitudeProvider attitudeProvider = npc.getRole().getAttitudeProvider();

// Get references to the source NPC and the target entity
Ref<EntityStore> self = world.getEntityRef(npc.getId());
Ref<EntityStore> target = world.getEntityRef(targetId);

// Calculate the attitude
Attitude currentAttitude = attitudeProvider.getAttitude(self, npc.getRole(), target, world.getComponentAccessor());

// Update the NPC's blackboard with this information
npc.getBlackboard().setAttitude(targetId, currentAttitude);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not add fields to an implementation that store data related to a specific NPC instance. This will cause state to be incorrectly shared among all NPCs using that provider.
- **Blocking Operations:** Do not perform database lookups, file I/O, or any other long-running blocking operations within getAttitude. Doing so will stall the AI tick for the responsible thread, potentially freezing many NPCs.
- **World Modification:** An IAttitudeProvider is a read-only query system. It must never modify the world state or entity components. All mutations must be handled by separate action-oriented systems that *use* the result of the attitude query.

## Data Pipeline
The IAttitudeProvider is a critical step in the AI's perception-to-action data flow. It transforms raw entity data into a high-level conceptual state (Attitude) that is easy for behavior trees to consume.

> Flow:
> AI Perception System detects a nearby entity -> AI Decision Node requests attitude evaluation -> **IAttitudeProvider.getAttitude** is called -> Provider reads FactionComponent, StateComponent, etc. from both entities -> Provider returns an Attitude object -> The Attitude is written to the NPC's Blackboard -> A Behavior Tree node reads the Attitude and selects an action (e.g., Attack, Flee, Greet).

