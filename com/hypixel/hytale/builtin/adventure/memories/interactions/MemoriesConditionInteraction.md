---
description: Architectural reference for MemoriesConditionInteraction
---

# MemoriesConditionInteraction

**Package:** com.hypixel.hytale.builtin.adventure.memories.interactions
**Type:** Behavior Node

## Definition
```java
// Signature
public class MemoriesConditionInteraction extends Interaction {
```

## Architecture & Concepts

The MemoriesConditionInteraction is a server-authoritative control flow node within the Hytale Interaction System. It functions as a conditional branch, analogous to a `switch` statement in traditional programming, but operating within a declarative, asset-driven behavior graph.

Its primary role is to query a global gameplay state—the current *Memories Level*—and redirect an entity's interaction sequence to a different subsequent Interaction based on that value. This allows for dynamic narrative or gameplay paths that adapt to world progression without requiring complex, hard-coded logic.

During the server's asset loading phase, the `compile` method is invoked. This transforms the high-level, map-based definition of branches into a more efficient, low-level series of operations using labels and jumps. The `OperationsBuilder` effectively flattens this conditional node and its children into a linear instruction set that the InteractionManager can execute with minimal overhead.

This component is fundamental for creating reactive and stateful adventure-mode content, where entity behaviors must change in response to major world events or player achievements tracked by the Memories system.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly via a constructor in game logic. They are deserialized from game asset files (e.g., JSON definitions for an NPC's behavior) by the engine's asset pipeline. The static `CODEC` field governs this process, which includes a critical `afterDecode` hook that populates transient, performance-oriented lookup tables.
- **Scope:** The definition of a MemoriesConditionInteraction is cached by the InteractionManager for the lifetime of the server session, or until assets are reloaded. An individual instance is effectively immutable after decoding. Its use is scoped to the execution of a specific interaction graph on an entity.
- **Destruction:** Instances are eligible for garbage collection once no active interaction contexts reference them. The cached definitions are purged on server shutdown.

## Internal State & Concurrency
- **State:** The component's state is immutable after deserialization. The primary fields, `next` (a map of memory levels to interaction names) and `failed` (a fallback interaction), are populated from the asset definition. The `sortedKeys` and `levelToLabel` fields are transient caches derived from `next` during the `afterDecode` phase. This pre-computation is a critical performance optimization, converting a potential O(N) search into an O(1) lookup at runtime.
- **Thread Safety:** This class is inherently thread-safe for all read operations, which constitute its entire runtime API. The design assumes that the `InteractionContext` object, which is passed into execution methods like `tick0`, is the sole carrier of mutable state and that its access is managed and synchronized by the calling `InteractionManager`. The MemoriesConditionInteraction itself holds no per-execution mutable state.

## API Surface

The public contract is defined by its overrides of the base Interaction class. Direct invocation is reserved for the InteractionManager.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(1) | **Server-side execution.** Retrieves the world's memory level, performs a hash map lookup, and instructs the context to jump to the appropriate compiled label. This is the authoritative logic path. |
| simulateTick0(...) | void | O(1) | **Client-side prediction.** Executes a similar jump based on the memory level provided in the replicated server state. Its outcome is non-authoritative and may be corrected by the server. |
| compile(builder) | void | O(N) | **Asset-time compilation.** Translates the declarative map of outcomes into a low-level series of jump operations and labels. N is the number of possible memory level outcomes. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.Server`. This is a critical network hint indicating that a client cannot predict the outcome of this interaction and must wait for the server's instruction. |

## Integration Patterns

### Standard Usage

This component is not used directly in Java code by content developers. Instead, it is defined declaratively within an entity's behavior asset files. The InteractionManager automatically finds, decodes, and executes it when an entity's behavior graph reaches this node.

A conceptual asset definition might look like this:

```json
{
  "id": "npc_memory_dialogue_gate",
  "type": "MemoriesConditionInteraction",
  "Next": {
    "0": "asset:hytale:dialogue_intro",
    "1": "asset:hytale:dialogue_after_quest1",
    "5": "asset:hytale:dialogue_world_saved"
  },
  "Failed": "asset:hytale:dialogue_default_fallback"
}
```

The engine processes this definition and invokes the `tick0` method when an entity is directed to run the `npc_memory_dialogue_gate` interaction.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MemoriesConditionInteraction()`. This bypasses the `CODEC` deserialization and, most importantly, the `afterDecode` logic that builds the essential `levelToLabel` lookup table. An instance created this way will fail at runtime.
- **State Mutation:** Do not attempt to modify the `next` or `failed` fields after the object has been decoded by the asset manager. The internal lookup tables will become stale, leading to undefined and inconsistent branching behavior.
- **Client-Side Trust:** The result of `simulateTick0` is for prediction only. Because `getWaitForDataFrom` returns `Server`, all authoritative game logic must assume that the client's state can be wrong and will be corrected by the server's execution of `tick0`.

## Data Pipeline

This component acts as a router for the flow of execution control, not for data in a traditional sense.

> Flow:
> InteractionManager invokes `tick0` -> **MemoriesConditionInteraction** reads `memoriesLevel` from `MemoriesPlugin` -> A target `Label` is selected via the internal `levelToLabel` map -> The `InteractionContext` is instructed to `jump` to the new Label -> Execution continues at the next Interaction node.

