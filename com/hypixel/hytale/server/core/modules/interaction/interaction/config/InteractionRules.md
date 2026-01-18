---
description: Architectural reference for InteractionRules
---

# InteractionRules

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Configuration Object

## Definition
```java
// Signature
public class InteractionRules implements NetworkSerializable<com.hypixel.hytale.protocol.InteractionRules> {
```

## Architecture & Concepts

The InteractionRules class is a data-driven configuration object that defines the behavioral contract for a server-side game interaction. It is the core component of the interaction arbitration system, responsible for resolving conflicts between simultaneous or sequential character actions. Its primary function is to answer two critical questions during gameplay:
1.  Can a new interaction start, given the currently active interactions?
2.  Should an active interaction be terminated because a new, higher-priority interaction has begun?

This class is not a service or a manager; it is a Plain Old Java Object (POJO) whose state is defined entirely by external configuration files (e.g., JSON assets). This design decouples the complex logic of interaction precedence from the core engine code, empowering game designers to tune gameplay mechanics without requiring new engine builds.

The static `CODEC` field is the most significant architectural feature. It uses the Hytale `BuilderCodec` system to deserialize asset files into strongly-typed InteractionRules instances. During this process, a critical performance optimization occurs in the `afterDecode` hook: string-based tags used for bypass rules are converted into integer indices via the `AssetRegistry`. This pre-computation allows the runtime validation logic to use fast integer comparisons instead of expensive string operations within the game loop.

As an implementation of `NetworkSerializable`, this class also serves as a bridge to the networking layer. It can be converted into a network-safe packet format to synchronize interaction behavior with the client, ensuring client-side predictions can accurately reflect server-authoritative rules.

## Lifecycle & Ownership

-   **Creation:** Instances are not instantiated directly in game logic. They are created by the engine's `Codec` system during the asset loading phase. The static `CODEC` field defines the deserialization mapping from a configuration file to the object's fields. A static `DEFAULT_RULES` instance exists as a fallback for interactions without explicit rules.
-   **Scope:** The lifetime of an InteractionRules object is bound to the parent `Interaction` asset it configures. It is loaded and cached in memory when its parent asset is loaded and persists for the server's entire runtime or until assets are reloaded.
-   **Destruction:** The object is eligible for garbage collection when its parent `Interaction` asset is unloaded from the `AssetRegistry`. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** The object's state is mutable upon creation. It is populated by the `Codec` system during deserialization, and its `...BypassIndex` fields are subsequently mutated by the `afterDecode` hook. After this initial loading and processing phase, the object should be treated as **effectively immutable**. Any runtime modification is a severe anti-pattern.
-   **Thread Safety:** This class is **not thread-safe** for mutation. It contains no locks or other synchronization primitives. All instantiation and state-setting occurs within the single-threaded context of the asset loading system. It is, however, safe for concurrent reads from multiple threads (e.g., entity processing threads in the game loop) once it has been fully initialized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validateInterrupts(...) | boolean | O(k) | Determines if this interaction should interrupt, or be interrupted by, another. Complexity is relative to the number of tags (k) on the entities. |
| validateBlocked(...) | boolean | O(k) | Determines if this interaction is blocked by, or is blocking, another. Complexity is relative to the number of tags (k) on the entities. |
| toPacket() | protocol.InteractionRules | O(N) | Serializes the rules into a network packet object for client synchronization. Complexity is relative to the number of rules (N). |

## Integration Patterns

### Standard Usage

InteractionRules are not used in isolation. They are retrieved from a parent `Interaction` object and used by a higher-level system, such as an `InteractionModule`, to arbitrate between two competing interactions.

```java
// Pseudo-code demonstrating conceptual usage within the engine
Interaction currentInteraction = entity.getActiveInteraction();
Interaction newInteraction = pendingInteractionRequest.getInteraction();

InteractionRules currentRules = currentInteraction.getRules();
InteractionRules newRules = newInteraction.getRules();

// Check if the new interaction is blocked by the current one
boolean isBlocked = newRules.validateBlocked(
    newInteraction.getType(),
    entity.getTags(),
    currentInteraction.getType(),
    entity.getTags(), // Assuming same entity for simplicity
    currentRules
);

if (!isBlocked) {
    // Proceed with starting the new interaction
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new InteractionRules()`. The object's state is complex and must be populated from an asset file via the engine's `Codec` system. Manual creation will result in an object with default or null fields, leading to incorrect behavior.
-   **Runtime Mutation:** Do not modify the fields of an InteractionRules object after it has been loaded. For example, `rules.blocking.add(someType)` during the game loop is not thread-safe and will cause unpredictable behavior across the server. All rule definitions must exist in the source asset files.
-   **Ignoring Bypass Indices:** Relying on the string-based `...Bypass` fields at runtime is inefficient. The core validation logic is designed to use the pre-calculated `...BypassIndex` integer fields for performance.

## Data Pipeline

The data for an InteractionRules object follows a clear path from design-time configuration to runtime execution.

> Flow:
> Game Asset (e.g., `swing_sword.json`) -> Hytale Codec System -> **InteractionRules** (Instantiation & Deserialization) -> `afterDecode` Hook (Populates Bypass Indices from AssetRegistry) -> Cached in Memory -> `InteractionModule` (Runtime Validation) -> `toPacket()` -> Network Layer -> Game Client

