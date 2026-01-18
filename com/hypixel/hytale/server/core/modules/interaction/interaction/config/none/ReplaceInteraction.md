---
description: Architectural reference for ReplaceInteraction
---

# ReplaceInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.none
**Type:** Configuration Object

## Definition
```java
// Signature
public class ReplaceInteraction extends Interaction {
```

## Architecture & Concepts
The **ReplaceInteraction** class is a control-flow mechanism within the server's interaction system. It does not represent a tangible action in the game world, such as attacking or using an item. Instead, it functions as a dynamic router or a conditional branch, redirecting the flow of execution to another interaction based on runtime variables.

Its primary role is to decouple interaction chains from static definitions, allowing for stateful and context-aware behaviors. This is achieved by looking up a key (the *variable* field) within the current **InteractionContext**. The value associated with that key is interpreted as the ID of the next **RootInteraction** to execute.

This pattern is fundamental for implementing complex, non-linear systems like quests, branching dialogues, or multi-stage crafting processes. For example, interacting with a specific block could trigger a **ReplaceInteraction**. If the player's context contains the variable *quest_stage: "complete"*, this interaction can route execution to a *quest_reward* interaction. If the variable is absent, it can route to a *default_dialogue* interaction.

**WARNING:** This interaction is a terminal node in its own execution path. It immediately sets its own state to *Finished* and triggers a new interaction execution via **InteractionContext.execute**. Any subsequent interactions in the original chain will be ignored.

## Lifecycle & Ownership
- **Creation:** Instances of **ReplaceInteraction** are not created programmatically during gameplay. They are deserialized from configuration files (e.g., JSON or HOCON) at server startup by the static **CODEC** field. This process populates the object's fields based on the asset definition.
- **Scope:** Once loaded, these objects are held in a central, server-wide registry of interactions, managed by systems like **RootInteraction**. They are effectively immutable singletons for the duration of the server session.
- **Destruction:** The objects are destroyed and garbage collected only when the server shuts down and the interaction registries are cleared.

## Internal State & Concurrency
- **State:** An instance of **ReplaceInteraction** is immutable after its creation by the codec. Its fields—*defaultValue*, *variable*, and *defaultOk*—are configured once and are not modified during runtime. The class itself is stateless.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. However, its methods operate on a mutable **InteractionContext** object. All operations involving this class must be synchronized externally, typically by ensuring they are only called from the main server thread which owns the game state and context objects.

## API Surface
The primary contract is defined by its parent **Interaction** class. The core logic is contained within the tick methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | void | O(1) | Server-side execution logic. On the first run, it performs the replacement logic by calling doReplace. |
| simulateTick0(...) | void | O(1) | Client-side prediction logic. Mirrors the server-side behavior for responsive feedback. |
| walk(...) | boolean | O(N) | Traverses the potential interaction paths for dependency analysis or validation, where N is the number of interactions in the target chain. |

## Integration Patterns

### Standard Usage
This interaction is defined in an asset file and triggered as part of a larger interaction sequence. The calling system is responsible for populating the **InteractionContext** with the necessary variables beforehand.

Conceptual configuration (e.g., in a JSON asset):
```json
{
  "id": "check_quest_status",
  "type": "ReplaceInteraction",
  "Var": "current_quest_node",
  "DefaultValue": "generic_npc_greeting",
  "DefaultOk": true
}
```

Execution flow in game logic:
```java
// Assume 'player' is the interacting entity and 'context' is their interaction context.
// A system (e.g., QuestManager) sets a variable based on player progress.
context.getInteractionVars().put("current_quest_node", "start_boss_fight");

// The game now triggers an interaction that leads to the "check_quest_status" interaction.
// The ReplaceInteraction will read "current_quest_node" and execute "start_boss_fight".
InteractionManager.getInstance().tick(context);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ReplaceInteraction()`. The object will be in an invalid state as its fields are not initialized. Always define interactions in configuration files to be loaded by the server's codec system.
- **Missing Context Variables:** Triggering this interaction without first setting the expected variable in the **InteractionContext** will cause it to use the *defaultValue*. If no *defaultValue* is provided and *defaultOk* is false, the system will log a severe error and the interaction will fail. This can halt complex gameplay sequences.
- **Expecting a World Action:** Do not use this class with the expectation that it will directly modify the world. It is a control-flow primitive only. Its sole purpose is to redirect to another interaction.

## Data Pipeline
The **ReplaceInteraction** acts as a conditional fork in the data flow of the interaction system. It consumes state from the **InteractionContext** and redirects the execution controller to a new interaction chain.

> Flow:
> InteractionManager Ticks -> **ReplaceInteraction.tick0** -> Reads **InteractionContext.getInteractionVars()** -> Resolves next Interaction ID -> **InteractionContext.execute(nextInteraction)** -> InteractionManager Ticks Next Interaction

