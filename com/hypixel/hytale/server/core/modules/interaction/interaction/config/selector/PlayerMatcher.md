---
description: Architectural reference for PlayerMatcher
---

# PlayerMatcher

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration Component

## Definition
```java
// Signature
public class PlayerMatcher extends SelectInteraction.EntityMatcher {
```

## Architecture & Concepts
The PlayerMatcher is a concrete implementation of the Strategy Pattern, designed to act as a specific rule within the server-side Interaction System. Its sole purpose is to identify if a target entity is a player.

This component is not intended for direct use in procedural game logic. Instead, it is a data-driven building block, instantiated by the server's configuration loader via its static CODEC field. This allows game designers to define complex interaction behaviors in external files (e.g., JSON) by specifying that a particular interaction should only apply to players. For example, a "Talk" interaction on an NPC might use a PlayerMatcher to ensure only players can trigger it.

The class operates on the server's core entity component system, using a CommandBuffer to inspect the archetype of a target entity. The `toPacket` method is a critical bridge to the client, translating this server-side rule into a lightweight network object. This allows the client to perform predictive checks or update UI elements (e.g., highlighting a valid target) without a full server round-trip.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the BuilderCodec system during server startup or when interaction configuration files are loaded. It is defined declaratively in configuration assets, not imperatively in code.
- **Scope:** The object is stateless and effectively immutable. Its lifetime is bound to the parent configuration object that references it, such as a specific `SelectInteraction`. It persists as long as the server's interaction configurations are held in memory.
- **Destruction:** Marked for garbage collection when the server shuts down or performs a full configuration reload that discards the old configuration objects.

## Internal State & Concurrency
- **State:** PlayerMatcher is a stateless object. It contains no mutable fields and its behavior is determined entirely by the arguments passed to its methods.
- **Thread Safety:** This class is inherently thread-safe. The `test0` method is a pure function with no side effects on the PlayerMatcher instance itself. It can be safely invoked by multiple worker threads processing game world events concurrently without requiring any locks or synchronization.

## API Surface
The public contract is minimal, focusing on the test condition and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CODEC | static BuilderCodec | N/A | **Critical.** Defines the serialization contract for loading this component from configuration files. |
| test0(attacker, target, commandBuffer) | boolean | O(1) | The core matching logic. Returns true if the target entity's archetype contains the Player component. |
| toPacket() | EntityMatcher | O(1) | Serializes the matcher into a network-transferable packet, setting the type to Player. |

## Integration Patterns

### Standard Usage
This class is not instantiated or called directly by feature developers. It is declared within a larger interaction configuration file. The system then invokes its `test0` method internally when an interaction event occurs.

A conceptual configuration might look like this:

```json
// Example: Defining an interaction that only targets players
{
  "id": "special_blessing",
  "selector": {
    "type": "hytale:player" // This "type" maps to the PlayerMatcher.CODEC
  },
  "effects": [ ... ]
}
```

The internal system usage would resemble the following:

```java
// How the InteractionModule would use an instance of this class
// Note: `matcher` is loaded from configuration, not created with `new`.
SelectInteraction.EntityMatcher matcher = loadedInteraction.getSelector();

// When an entity tries to interact with a target...
boolean isMatch = matcher.test0(attackerRef, targetRef, commandBuffer);
if (isMatch) {
    // Apply interaction effects
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PlayerMatcher()`. The object's identity and functionality are tied to the configuration system that uses its static CODEC. Direct instantiation bypasses this entire system and will lead to non-functional interactions.
- **Extending for Custom Logic:** Do not extend this class to match other entity types. Instead, create a new, separate matcher class with its own CODEC. This class is purpose-built and should remain sealed in its function.

## Data Pipeline
The primary data flow for PlayerMatcher is part of the server's configuration loading process, not a real-time event stream.

> Flow:
> Server Startup -> Configuration Asset (e.g., JSON file) -> **BuilderCodec** -> Instantiated **PlayerMatcher** -> Stored in memory as part of an Interaction Rule -> Game Event Occurs -> Interaction System invokes **PlayerMatcher.test0(target)** -> Boolean Result -> Game Logic Branch (Interaction Allowed/Denied)

