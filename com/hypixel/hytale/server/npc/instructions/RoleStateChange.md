---
description: Architectural reference for RoleStateChange
---

# RoleStateChange

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Callback Interface

## Definition
```java
// Signature
public interface RoleStateChange {
```

## Architecture & Concepts
The RoleStateChange interface is a behavioral contract that defines a set of callbacks for key lifecycle events within an NPC's Role component. It functions as a core component of the server-side NPC event system, embodying the Observer design pattern.

Systems that need to react to changes in an NPC's state—such as quest managers, AI behavior trees, or zone controllers—implement this interface. The associated Role object maintains a collection of these listeners and invokes the appropriate methods synchronously when a state transition occurs.

The use of default methods is a critical design choice. It allows implementers to selectively override only the event handlers they are interested in, creating a clean, non-intrusive integration point without the need for abstract adapter classes. This interface is the primary mechanism for decoupling high-level game logic from the low-level mechanics of NPC entity management.

## Lifecycle & Ownership
As an interface, RoleStateChange does not have its own lifecycle. The lifecycle pertains to the objects that *implement* this interface.

-   **Creation:** Concrete implementations are instantiated by various game systems that require insight into NPC behavior. For example, a QuestManager might create a specific listener to track a quest-critical NPC.
-   **Scope:** The lifetime of a listener object is determined by its owner. To receive events, it must be explicitly registered with a specific Role instance. It will receive callbacks for as long as it remains registered and the Role exists.
-   **Destruction:** The listener is eligible for garbage collection once its owning system releases its reference *and* it has been unregistered from the Role.

**WARNING:** Failure to unregister a listener from a long-lived Role can result in a memory leak. The Role will maintain a strong reference to the listener, preventing it from being garbage collected.

## Internal State & Concurrency
-   **State:** This interface is inherently stateless. All state is managed by the concrete classes that implement it.
-   **Thread Safety:** The interface itself provides no concurrency guarantees. Callbacks are invoked synchronously on the main server thread responsible for NPC updates.

**WARNING:** Implementations of RoleStateChange **must be non-blocking**. Any long-running operations, such as network requests, file I/O, or complex computations, will directly stall the server's NPC processing loop, leading to severe performance degradation. Such tasks must be offloaded to a separate worker thread or a job queue.

## API Surface
The API consists entirely of callback methods triggered by the NPC system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role) | void | O(N) | Invoked when the listener is first registered, allowing for initial setup. |
| motionControllerChanged(...) | void | O(N) | Fired when the NPC's underlying MotionController is swapped or reconfigured. |
| loaded(Role) | void | O(N) | Fired when the NPC's data is loaded into memory, typically before it is spawned into the world. |
| spawned(Role) | void | O(N) | Fired when the NPC entity is physically added to a world and becomes active. |
| unloaded(Role) | void | O(N) | Fired when the NPC is removed from the world and its data is about to be persisted or discarded. |
| removed(Role) | void | O(N) | Fired when the NPC entity is permanently deleted from the game. |
| teleported(Role, World, World) | void | O(N) | Fired immediately after an NPC is teleported between worlds or to a distant location in the same world. |

*Complexity is O(N) where N is the number of registered listeners for a given Role.*

## Integration Patterns

### Standard Usage
A system implements the interface, overrides the desired methods, and registers an instance with the target NPC's Role component.

```java
// A simple listener to activate a quest when a specific NPC spawns.
public class QuestStartListener implements RoleStateChange {
    private final QuestId targetQuest;

    public QuestStartListener(QuestId quest) {
        this.targetQuest = quest;
    }

    @Override
    public void spawned(Role role) {
        // This logic is executed on the main server thread.
        // Keep it fast and non-blocking.
        NPCEntity npc = role.getNpc();
        System.out.println("NPC " + npc.getId() + " has spawned. Starting quest: " + targetQuest);
        
        // Example: Get a service and trigger quest logic
        // QuestService service = GameContext.get().getQuestService();
        // service.beginQuestForNearbyPlayers(targetQuest, npc.getPosition());
    }
}

// In a QuestManager or similar system:
NPCEntity targetNpc = world.findNpcById(...);
if (targetNpc != null) {
    Role npcRole = targetNpc.getRole();
    QuestStartListener listener = new QuestStartListener(QUEST_ID_FETCH_THE_ARTIFACT);
    
    // The assumed API for registering the listener
    npcRole.addStateChangeListener(listener);
}
```

### Anti-Patterns (Do NOT do this)
-   **Blocking Operations:** Do not perform database queries, file access, or network calls directly inside a callback. This will freeze the NPC simulation.
-   **Complex State Management:** Avoid creating complex, stateful listeners. Prefer simple, single-purpose listeners. If complex logic is needed, the listener should delegate to a more robust state machine or service.
-   **Dangling Listeners:** Do not forget to unregister listeners when they are no longer needed. For example, if the quest system that owns the listener is shut down, it must iterate through its active listeners and unregister them from their respective Roles to prevent memory leaks.

## Data Pipeline
RoleStateChange acts as a notification endpoint rather than a data processing stage. It is a "tap" into the NPC lifecycle event stream.

> **Flow:**
> Server Engine Event (e.g., Chunk Load, Player Command) -> EntityStore Update -> `Role` Component State Change -> **RoleStateChange Callback Invocation** -> Custom Game System Logic (e.g., Quest Update, AI Trigger)

