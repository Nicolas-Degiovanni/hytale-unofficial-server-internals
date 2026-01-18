---
description: Architectural reference for UseEntityInteraction
---

# UseEntityInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Handler Object

## Definition
```java
// Signature
public class UseEntityInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The UseEntityInteraction class is a server-side handler for a generic, client-initiated action to "use" a target entity. It functions as a critical *dispatcher* within the server's Interaction System, translating a low-level network request into a specific, high-level gameplay interaction.

Its primary architectural role is to decouple the client's intent from the target entity's behavior. The client does not need to know *what* using an entity does; it only sends a standard request to use it. This class receives that request on the server and queries the target entity's **Interactions** component to determine the appropriate subsequent interaction to execute.

This pattern allows game designers to define complex entity behaviors (e.g., talking to an NPC, opening a chest, attacking a monster) in data files by configuring the entity's component, without requiring any changes to the client-server protocol or this handler class. All state modifications are performed through a **CommandBuffer**, ensuring that changes are safely queued and applied within the server's main game tick.

## Lifecycle & Ownership
-   **Creation:** Instances of UseEntityInteraction are not created directly via a constructor in gameplay code. They are deserialized from game asset definitions (e.g., JSON configuration files) at server startup by the engine's asset loading pipeline, which uses the public static **CODEC** field.
-   **Scope:** An instance of this class is a stateless, shared definition. It persists for the entire server session once loaded. It is effectively a singleton in behavior, though multiple definitions could exist if configured.
-   **Destruction:** The object is garbage collected when the server shuts down and all asset definitions are unloaded.

## Internal State & Concurrency
-   **State:** This class is **immutable and stateless**. It holds no instance fields that are modified during execution. All necessary state for an operation (e.g., the target entity, the instigating player) is provided via the **InteractionContext** parameter in the `firstRun` method.

-   **Thread Safety:** The class itself is inherently thread-safe. However, it operates on stateful, non-thread-safe objects like **InteractionContext** and **CommandBuffer**.

    **WARNING:** All method calls on a UseEntityInteraction instance that involve an InteractionContext must be strictly confined to the server's main game loop thread to prevent race conditions and data corruption within the Entity-Component-System.

## API Surface
The public contract is primarily defined by its inherited methods, which are invoked by the Interaction System.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldown) | void | O(1) | Core dispatch logic. Resolves and executes a specific interaction based on the target entity's component data. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Declares that this interaction requires data from the client before execution can begin. |
| generatePacket() | Interaction | O(1) | Constructs the network packet that a client would send to initiate this interaction. |
| needsRemoteSync() | boolean | O(1) | Returns true, flagging this as a client-initiated action that must be synchronized with the server. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly by developers. Instead, it is triggered automatically by the server's Interaction System when a client sends a corresponding `UseEntityInteraction` packet. The system is configured through entity component data.

A typical scenario involves an entity's asset definition:

```json
// Example pseudo-JSON for an entity's component
{
  "component": "Interactions",
  "interactions": {
    "PRIMARY": "hytale:talk_to_npc",
    "SECONDARY": "hytale:trade_with_npc"
  }
}
```
When the client "uses" this entity, the server invokes `UseEntityInteraction.firstRun`. This method reads the `PRIMARY` interaction ID ("hytale:talk_to_npc") from the component and executes that new interaction.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new UseEntityInteraction()`. The system relies on the definitions loaded via the **CODEC**. Manually creating an instance bypasses the asset system and will likely fail.
-   **Stateful Implementation:** Do not extend this class and add mutable instance fields. The shared, stateless nature of interaction handlers is a core architectural principle.
-   **Bypassing the Context:** Do not attempt to access the **EntityStore** or other world state globally. All interactions must flow through the provided **InteractionContext** and **CommandBuffer** to ensure transactional integrity.

## Data Pipeline
The flow of data for a "use entity" event is unidirectional, starting from client input and resulting in a server-side state change.

> Flow:
> Client Input (e.g., Right-Click) -> Client sends `UseEntityInteraction` packet with `entityId` -> Server Network Layer decodes packet -> Interaction System is notified -> **UseEntityInteraction.firstRun()** is invoked with context -> Entity's `Interactions` component is read -> A new, specific interaction (e.g., `TalkToNpcInteraction`) is executed via `context.execute()` -> The new interaction's logic is processed by the game loop.

