---
description: Architectural reference for PrefabSetAnchorInteraction
---

# PrefabSetAnchorInteraction

**Package:** com.hypixel.hytale.builtin.buildertools.prefabeditor
**Type:** Transient Handler

## Definition
```java
// Signature
public class PrefabSetAnchorInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The PrefabSetAnchorInteraction class is a concrete implementation of the server's **Interaction System**. It encapsulates the specific server-side logic for a single, discrete player action: setting the anchor point of a prefab within a build tool editing session.

This class operates as a stateless, command-like object. It is designed to be instantiated by the engine in direct response to a player's primary or secondary interaction (e.g., a mouse click) when using a configured tool. Its primary architectural role is to act as a bridge between raw player input and the stateful `PrefabEditSessionManager`, which is the central service tracking prefab editing state on a per-player basis.

A key design aspect is its data-driven nature, indicated by the static `CODEC` field. This allows game designers to associate this interaction logic with specific items or tools through configuration files, completely decoupling the game logic from the item definitions.

### Lifecycle & Ownership
- **Creation:** Instantiated on-demand by the server's Interaction Module. The engine uses the `CODEC` to deserialize a definition from game data and construct an instance when a player performs an action with an item configured to use this interaction. It is never created via a direct `new` call in gameplay code.
- **Scope:** The object's lifetime is extremely short, scoped precisely to the execution of a single `firstRun` method call. It is created, used for one operation, and immediately becomes eligible for garbage collection.
- **Destruction:** Handled automatically by the Java Garbage Collector after the `firstRun` method completes. It holds no persistent references and is not registered with any long-lived services.

## Internal State & Concurrency
- **State:** **Stateless and Immutable**. This class contains no instance fields. Its behavior is determined entirely by the arguments passed to the `firstRun` method, primarily the `InteractionContext`. All state modification is performed on external systems, such as the `PrefabEditSession` and the world via the `CommandBuffer`.

- **Thread Safety:** **Conditionally Thread-Safe**. The class itself has no mutable state, making it inherently safe. However, its `firstRun` method operates on a `CommandBuffer` and interacts with the `PrefabEditSessionManager`. The safety of an invocation is therefore dependent on the thread-safety guarantees of the surrounding Entity Component System. It is designed and expected to be executed exclusively on the main server thread corresponding to the player's world.

## API Surface
The public contract is minimal, consisting only of the overridden method from its parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | void | O(N) | Executes the anchor-setting logic. N is the number of loaded prefabs in the player's session. Mutates the player's `PrefabEditSession` and queues world changes via the `CommandBuffer`. Throws `AssertionError` if the context is malformed. |

## Integration Patterns

### Standard Usage
This class is not invoked directly by developers. It is configured in game data (e.g., JSON or HOCON files) and triggered automatically by the server's Interaction Module. The following example is conceptual, demonstrating the context in which the engine uses this class.

```java
// CONCEPTUAL: How the engine uses this class.
// A developer does not write this code.

// 1. An item's definition points to this interaction's CODEC.
// 2. A player with an active PrefabEditSession right-clicks a block with that item.
// 3. The engine creates the context and instantiates the handler.
InteractionContext context = createInteractionContextForPlayer(...);
PrefabSetAnchorInteraction interaction = codec.decode(...); // Engine decodes from config

// 4. The engine invokes the handler, triggering the core logic.
interaction.firstRun(InteractionType.Primary, context, cooldownHandler);

// 5. As a result, the player's session state is mutated.
PrefabEditSession session = sessionManager.getPrefabEditSession(playerUuid);
// The session's selected prefab and its anchor point are now updated.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PrefabSetAnchorInteraction()` in gameplay code. The engine's data-driven systems are solely responsible for its creation and lifecycle. Doing so bypasses the entire configuration system.
- **Stateful Implementation:** Do not modify this class to hold instance state (e.g., caching the player's UUID). It must remain stateless to function correctly and predictably within the interaction system, which may reuse decoded instances.
- **Bypassing the CommandBuffer:** All world modifications *must* be routed through the `CommandBuffer` provided in the `InteractionContext`. Direct modification of world state from this class will break transactional integrity and cause severe concurrency issues.

## Data Pipeline
The flow of data for a successful anchor-setting operation begins with player input and ends with both a server-side state change and a client-side visual update.

> Flow:
> Player Input (Click) -> Network Packet -> Server Interaction Module -> **PrefabSetAnchorInteraction.firstRun()** -> PrefabEditSessionManager (State Update) -> CommandBuffer (Queue World Changes) -> Network Packet (Anchor Highlight) -> Client Render

