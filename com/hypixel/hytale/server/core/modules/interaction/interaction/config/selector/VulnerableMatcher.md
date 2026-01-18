---
description: Architectural reference for VulnerableMatcher
---

# VulnerableMatcher

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration Object / Predicate

## Definition
```java
// Signature
public class VulnerableMatcher extends SelectInteraction.EntityMatcher {
```

## Architecture & Concepts
The VulnerableMatcher is a stateless predicate component within the server-side Interaction System. Its sole function is to act as a filter, identifying entities that are susceptible to damage or other aggressive interactions. It is not a standalone service but rather a declarative rule that is composed into a larger interaction definition.

Architecturally, it serves as a bridge between the declarative configuration of game mechanics and the server's Entity Component System (ECS). When an interaction is triggered, the system evaluates a chain of matchers to validate the target. The VulnerableMatcher performs this validation by querying the target entity's archetype for the presence of the **Invulnerable** component.

The inclusion of the `toPacket` method indicates that this server-side logic has a corresponding representation on the client. The server serializes this rule into a network packet, allowing the client to perform predictive validation or update UI elements, such as crosshairs, without a full server round-trip.

## Lifecycle & Ownership
- **Creation:** Instances are exclusively created by the Hytale Codec system during the deserialization of game configuration assets. The static **CODEC** field is the designated factory. Direct instantiation is a critical anti-pattern.
- **Scope:** The object's lifetime is bound to the parent interaction configuration that contains it. It is effectively immutable and persists as long as the game mechanic it defines is loaded in memory.
- **Destruction:** The object is marked for garbage collection when its parent configuration is unloaded, typically during a world shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no instance fields and its behavior is purely a function of the arguments passed to its methods.
- **Thread Safety:** The VulnerableMatcher is inherently **thread-safe**. Its stateless nature ensures that the `test0` method can be invoked concurrently from multiple threads without risk of race conditions or data corruption. The responsibility for thread-safe management of the `EntityStore` and `CommandBuffer` arguments lies with the calling system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test0(attacker, target, commandBuffer) | boolean | O(1) | Evaluates if the target entity lacks the Invulnerable component. Returns true if the entity is vulnerable, false otherwise. |
| toPacket() | EntityMatcher | O(1) | Serializes this rule into a network-optimized packet for client-side synchronization. |

## Integration Patterns

### Standard Usage
This component is not intended for direct, imperative use in gameplay code. It is designed to be declared within a data file that defines an interaction. The engine's configuration loader then instantiates it.

A conceptual configuration might look like this:
```yaml
# Example: Defining a "Basic Attack" interaction in a game asset file
interaction:
  type: "SelectInteraction"
  targetSelector:
    matchers:
      - type: "VulnerableMatcher" # The engine uses this 'type' to find the correct CODEC
      - type: "LineOfSightMatcher"
  onSuccess:
    - action: "ApplyDamage"
      amount: 10
```

The engine internally invokes the matcher during the interaction validation phase:
```java
// Engine-level code, NOT for typical gameplay programming
SelectInteraction interaction = loadInteractionFromConfig("BasicAttack");
boolean canInteract = interaction.getTargetSelector().test(attackerRef, targetRef, commandBuffer);

if (canInteract) {
    // Execute interaction logic
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new VulnerableMatcher()`. The object is not designed for manual creation and bypasses the critical codec system responsible for its lifecycle.
- **Extending for Custom Logic:** Do not extend this class to add new matching logic. Instead, create a new, separate implementation of `SelectInteraction.EntityMatcher` with its own `CODEC`.

## Data Pipeline
The VulnerableMatcher acts as a gate in the server's interaction processing pipeline.

> Flow:
> Player Input (Attack) -> Interaction System -> Target Selection -> **VulnerableMatcher.test0(target)** -> [Filter Pass] -> Execute Interaction Effects (e.g., Damage)
>
> Flow (Client Sync):
> Server Asset Load -> **VulnerableMatcher.toPacket()** -> Network Packet -> Client Interaction System -> Predictive UI Update (e.g., Crosshair Color)

