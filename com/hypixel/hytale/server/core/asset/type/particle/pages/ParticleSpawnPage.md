---
description: Architectural reference for ParticleSpawnPage
---

# ParticleSpawnPage

**Package:** com.hypixel.hytale.server.core.asset.type.particle.pages
**Type:** Transient

## Definition
```java
// Signature
public class ParticleSpawnPage extends InteractiveCustomUIPage<ParticleSpawnPage.ParticleSpawnPageEventData> {
```

## Architecture & Concepts
The ParticleSpawnPage is a server-side controller that powers a client-side user interface for selecting and spawning particle effects. It is a concrete implementation of the server-driven UI framework, extending InteractiveCustomUIPage to manage a dynamic, stateful screen for a specific player.

This class acts as a bridge between player input events, the server's UI command system, and the core Entity Component System (ECS). When a player interacts with the UI (e.g., searching for a particle, selecting it, or clicking spawn), the client sends a serialized event to the server. The ParticleSpawnPage instance associated with that player receives this event via the handleDataEvent method, processes it, and then orchestrates changes in both the UI and the game world.

Key responsibilities include:
-   Dynamically building and updating a list of available particle systems based on asset definitions.
-   Handling real-time search queries using a fuzzy string matching algorithm to filter the list.
-   Managing a temporary "preview" entity within the world's EntityStore to give the player a visual representation of the selected particle effect.
-   Responding to the final "spawn" command by creating a persistent particle effect in the world and cleaning up the preview entity.

This component is fundamentally a state machine whose state is manipulated by client-side events and whose transitions result in commands sent back to the client or modifications to the server's world state.

### Lifecycle & Ownership
-   **Creation:** An instance is created by the server's UI management system when a player is shown this specific page, typically triggered by an in-game command or another UI interaction. The constructor requires a PlayerRef, tightly coupling the page's lifecycle to a specific player session.
-   **Scope:** The object persists only as long as the player has the particle spawn UI open. It is a short-lived, session-specific component.
-   **Destruction:** The onDismiss method is invoked by the framework when the player closes the UI. This method is the designated cleanup point and is responsible for removing the preview entity from the EntityStore to prevent orphaned entities. The Java object itself is then eligible for garbage collection.

## Internal State & Concurrency
-   **State:** This class is highly stateful and mutable. It maintains the current search query, the complete list of particle system IDs, the currently selected particle ID, and a reference (Ref) to the preview entity in the EntityStore. It also stores the calculated position and rotation for spawning. This state is modified exclusively in response to events from the associated player.
-   **Thread Safety:** This class is **not thread-safe** and must be accessed only from the main server thread responsible for the world in which the player exists. All methods that interact with the Store (the ECS world) assume they are executing within a synchronized game tick. Unmanaged multi-threaded access will lead to severe concurrency issues, including data corruption in the EntityStore and network desynchronization.

## API Surface
The public contract is defined by the InteractiveCustomUIPage inheritance, focusing on lifecycle and event handling.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N log N) | Builds the initial UI state, sending commands to the client. Complexity is dominated by sorting N particle system names. |
| handleDataEvent(ref, store, data) | void | O(M) | Primary event handler for all client interactions. Complexity varies: search is O(M) due to fuzzy matching over M assets; selection and spawning are closer to O(1). |
| onDismiss(ref, store) | void | O(1) | Lifecycle callback for cleanup. Removes the preview entity from the world. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation by developers. It is managed by the server's UI framework. A developer would typically define a command or trigger that instructs the framework to open this page for a given player.

```java
// Hypothetical example of a command triggering the page
// This code would exist within a command handler, not in general game logic.

PlayerRef player = ...;
player.getUIManager().openPage(new ParticleSpawnPage(player));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use new ParticleSpawnPage() outside of a designated UI management or command-handling context. The page's lifecycle is tied to the player's UI state and must be managed by the server framework.
-   **Cross-Thread Access:** Never call methods on a ParticleSpawnPage instance from a thread other than the one managing the player's world. Doing so will corrupt the EntityStore.
-   **State Tampering:** Do not manually modify the internal state (like selectedParticleSystemId or particleSystemPreview) from outside the class. All state changes must flow through the handleDataEvent method to ensure UI and world consistency.
-   **Ignoring onDismiss:** Failing to ensure the UI framework calls onDismiss upon page closure will result in orphaned preview entities that persist indefinitely in the game world.

## Data Pipeline
The flow of data for this component is a closed loop between the client, the server-side page controller, and the game world.

> **Client Input Flow:**
> Client UI Interaction (e.g., typing in search) -> Client sends CustomUIEvent Packet -> Server Network Layer -> **ParticleSpawnPage.handleDataEvent** -> Logic Execution (e.g., filtering list) -> UICommandBuilder populated -> Server sends UIUpdate Packet -> Client UI Renders Changes

> **World Interaction Flow (Spawn Action):**
> Client clicks "Spawn" -> CustomUIEvent Packet -> **ParticleSpawnPage.handleDataEvent** -> `store.removeEntity` (for preview) -> `ParticleUtil.spawnParticleEffect` -> `store.addEntity` (for final effect) -> Entity is created in the world -> Entity state is replicated to nearby clients

