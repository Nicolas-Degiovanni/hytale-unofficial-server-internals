---
description: Architectural reference for OpenBenchPageInteraction
---

# OpenBenchPageInteraction

**Package:** com.hypixel.hytale.builtin.crafting.interaction
**Type:** Configuration Object

## Definition
```java
// Signature
public class OpenBenchPageInteraction extends SimpleBlockInteraction {
```

## Architecture & Concepts
The OpenBenchPageInteraction class is a server-side, data-driven command that links a player's interaction with a block to the opening of a specific crafting user interface. It is a concrete implementation within the server's Interaction System, designed to be a self-contained and reusable piece of logic.

Its primary role is to act as a bridge between a physical world action (interacting with a block) and a player-centric UI state change (opening a crafting bench). The class is not a manager or service; it is a stateless definition of an *effect*.

The design is highly declarative. An instance of this class, configured with a specific PageType, describes *what* should happen when triggered. The broader Interaction Module is responsible for invoking this logic with the correct context when a player interacts with a configured block. The presence of a static CODEC field is a critical architectural indicator, signifying that these interactions are defined in external asset files (e.g., JSON) and loaded at runtime. This allows game designers to configure complex block behaviors without modifying engine source code.

## Lifecycle & Ownership
- **Creation:** Instances are primarily created by the framework's deserialization engine (the CODEC) during server startup as it loads game assets. The static instances, such as SIMPLE_CRAFTING, are initialized at class-loading time and serve as built-in, default interaction definitions.
- **Scope:** An instance of OpenBenchPageInteraction is stateless and reusable. Once loaded from configuration, it typically persists for the entire server session, held within a central interaction registry or directly by block type definitions. It is not tied to a specific player or world instance.
- **Destruction:** Instances are reclaimed by the Java Virtual Machine during server shutdown or a hot-reload of game assets. There is no manual destruction or cleanup logic associated with this class.

## Internal State & Concurrency
- **State:** The class has minimal internal state, consisting of the PageType enum. This field is set during construction or deserialization and is treated as immutable thereafter. The object holds no session-specific or world-specific data, making it a pure command object.
- **Thread Safety:** OpenBenchPageInteraction is inherently thread-safe. Its core method, interactWithBlock, is a pure function with respect to the object's state; it operates exclusively on method parameters which are scoped to a single, synchronous game tick. A single shared instance can be safely executed for any number of interactions without risk of race conditions.

## API Surface
The public contract is almost exclusively defined by the inherited `interactWithBlock` method. Direct construction is reserved for the framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| interactWithBlock(...) | void | O(1) | Server-side execution logic for the interaction. Validates that the target block is a valid crafting bench and that the player is in a state to open one. If successful, it dispatches a command to the player's PageManager to open the specific crafting window defined by the internal PageType. |

## Integration Patterns

### Standard Usage
This class is not intended to be invoked directly in gameplay code. It is configured declaratively in game data files and triggered automatically by the server's Interaction Module. The following is a conceptual example of how the system uses the class.

```java
// This is a conceptual example of how the Interaction System
// invokes this class. Developers do not call this method directly.

// 1. A player interacts with a block in the world.
// 2. The system resolves this action to a configured interaction.
OpenBenchPageInteraction interaction = interactionRegistry.get("*Diagram_Crafting_Default");

// 3. The system populates the context and invokes the interaction.
//    This happens within the server's main game loop.
interaction.interactWithBlock(
    world,
    commandBuffer,
    InteractionType.PRIMARY,
    interactionContext,
    player.getHeldItem(),
    targetBlockPosition,
    cooldownHandler
);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new OpenBenchPageInteraction()` in gameplay systems. Interactions must be defined in data assets and retrieved from the appropriate registry. Using the static instances like `OpenBenchPageInteraction.SIMPLE_CRAFTING` is acceptable for hardcoded logic.
- **Stateful Logic:** Do not extend this class to add player-specific or session-specific state. The design relies on these objects being stateless and reusable.
- **Client-Side Invocation:** The `interactWithBlock` method contains server-authoritative logic and **must not** be called on the client. The corresponding `simulateInteractWithBlock` method is intentionally empty, indicating this action has no client-side prediction.

## Data Pipeline
The flow of data for this interaction begins with player input and results in a UI update on the client, orchestrated entirely by the server.

> Flow:
> Player Input -> Network Packet (Interact) -> Server Interaction Module -> **OpenBenchPageInteraction.interactWithBlock** -> Player PageManager -> Command to open a new Window -> Network Packet (Open Window) -> Client UI Renders Window

