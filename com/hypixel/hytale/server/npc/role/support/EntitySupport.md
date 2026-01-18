---
description: Architectural reference for EntitySupport
---

# EntitySupport

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Transient

## Definition
```java
// Signature
public class EntitySupport {
```

## Architecture & Concepts

The EntitySupport class is a stateful helper object that serves as a direct extension of an NPCEntity's core logic. It is not an ECS component itself, but rather a high-level manager that orchestrates state and interacts with the underlying component system on behalf of its parent NPC.

Its primary architectural role is to decouple the complex, high-level NPC behavior logic (such as roles, instructions, and scripted expressions) from the low-level details of entity state management. It centralizes several key domains of NPC functionality:

*   **Motion Queuing:** It provides a single-slot buffer for the next body and head motion instructions. This prevents conflicting commands from the behavior system and ensures that motions are executed sequentially.
*   **State Management:** It manages transient NPC state that does not fit neatly into the ECS model, such as pending display name changes and lists of active player tasks.
*   **Delayed Execution:** It maintains and processes a list of components that are in a temporary waiting state, ticking them each frame until their delay expires. This is critical for implementing timed behaviors without blocking the main NPC logic.
*   **Scripting Interface:** Through its static factory method createScope, it provides a sandboxed environment (StdScope) for the NPC expression engine, exposing vital real-time entity data like health and obstruction status in a safe, read-only manner.

This class is a foundational piece of the server-side NPC framework, acting as the immediate "support" system for an NPC's active role.

### Lifecycle & Ownership

-   **Creation:** An EntitySupport instance is created during the initialization of an NPCEntity's role. The constructor requires the parent NPCEntity and a BuilderRole, indicating it is configured from asset definitions and bound tightly to a specific entity instance.
-   **Scope:** The lifecycle of an EntitySupport object is identical to that of its parent NPCEntity. It persists as long as the NPC exists within the game world.
-   **Destruction:** The object has no explicit destruction or cleanup method. It is marked for garbage collection when its parent NPCEntity is unloaded or destroyed.

## Internal State & Concurrency

-   **State:** EntitySupport is highly **mutable**. Its fields represent the immediate, transient state of the NPC, including queued instructions, pending name changes, and active timers. This state is read and modified continuously throughout the NPC's tick cycle.

-   **Thread Safety:** This class is **not thread-safe** and must be considered thread-hostile. All methods are designed to be called exclusively from the main server thread that manages the game loop. Unsynchronized access from other threads will lead to race conditions, instruction loss, and world state corruption. No internal locking mechanisms are used, as all access is assumed to be sequential.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(float dt) | void | O(N) | Processes all registered delaying components. N is the number of components currently in a delay state. |
| setNextBodyMotionStep(Instruction step) | boolean | O(1) | Attempts to queue the next body motion. Returns false if a motion is already queued, preventing instruction overwrites. |
| setNextHeadMotionStep(Instruction step) | boolean | O(1) | Attempts to queue the next head motion. Returns false if a motion is already queued. |
| nominateDisplayName(String displayName) | void | O(1) | Stages a display name change to be applied at a safe point in the tick cycle. |
| handleNominatedDisplayName(...) | void | O(1) | Applies the nominated display name to the parent entity's components. |
| registerDelay(IComponentExecutionControl) | void | O(1) | Adds a component to the internal list for processing during the tick. |
| createScope(NPCEntity entity) | StdScope | O(1) | **Static Factory.** Creates a sandboxed scope for the expression engine, pre-populated with NPC state suppliers. |

## Integration Patterns

### Standard Usage

EntitySupport is not intended for direct use by most systems. It is managed internally by an NPCEntity's Role. The Role's behavior logic (e.g., an Instruction executor) is the primary consumer of this class's API.

```java
// Within an NPC's behavior processing logic
// The 'support' object is a field within the NPC's Role
EntitySupport support = this.npc.getRole().getSupport();

// Queue a 'look at target' instruction
Instruction lookInstruction = createLookInstruction(target);
boolean wasQueued = support.setNextHeadMotionStep(lookInstruction);

if (!wasQueued) {
    // Handle the case where another instruction is already pending
    // This is a critical control flow check
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance using `new EntitySupport()`. The object is tightly coupled to the NPC's lifecycle and must be instantiated by the NPC's Role during its own initialization.
-   **Asynchronous Modification:** Do not access or modify an EntitySupport instance from any thread other than the main server thread. This will break the NPC's state machine.
-   **Ignoring Return Values:** The `setNext...MotionStep` methods return a boolean indicating success. Ignoring a `false` return value means your instruction was dropped, which can lead to cascading behavior failures.
-   **State Caching:** Do not cache the result of getters like `getNextBodyMotionStep`. The value is transient and will be cleared by the motion system after being consumed. Always fetch the value when it is needed.

## Data Pipeline

EntitySupport acts as a state buffer and command processor. It does not typically participate in a linear data pipeline but rather serves as a hub for state changes. The flow for a display name change provides a clear example of its role.

> Flow:
> NPC Behavior System -> `nominateDisplayName("Goblin Grunt")` -> **EntitySupport** (stores name internally) -> Server Tick -> `handleNominatedDisplayName()` -> `ComponentAccessor` -> ECS Update (`DisplayNameComponent`, `Nameplate`)

