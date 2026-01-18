---
description: Architectural reference for InteractionTarget
---

# InteractionTarget

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.util
**Type:** Utility

## Definition
```java
// Signature
public enum InteractionTarget {
```

## Architecture & Concepts
InteractionTarget is a foundational enum that provides a type-safe, declarative mechanism for specifying the subject of an action within the server's interaction system. In any given interaction—such as a player using an item, a script triggering an effect, or an entity taking damage—this enum is used to unambiguously identify which entity fills a specific role.

It abstracts the core roles in an event chain into three distinct constants:
*   **USER:** The entity that directly initiated the interaction. For example, the player who clicked a mouse button.
*   **OWNER:** The entity that possesses the object or context of the interaction. This can differ from the USER. For example, if a player activates a trap owned by another player, the USER is the activator and the OWNER is the trap's owner.
*   **TARGET:** The entity being acted upon. For example, the monster that is hit by a spell.

The inclusion of a static `CODEC` field signifies its role in data-driven design. This enum is intended to be serialized and deserialized from game configuration files (e.g., JSON or HOCON) or network packets, allowing designers and developers to define complex behaviors without writing hardcoded logic.

## Lifecycle & Ownership
- **Creation:** Instances of this enum (USER, OWNER, TARGET) are constants created by the Java Virtual Machine during class loading. They are not instantiated at runtime.
- **Scope:** As static constants, they exist for the entire lifetime of the server process.
- **Destruction:** The instances are garbage collected when the JVM shuts down. Their lifecycle is managed entirely by the JVM.

## Internal State & Concurrency
- **State:** InteractionTarget is **immutable**. Its instances are constants and hold no mutable state.
- **Thread Safety:** This enum is inherently **thread-safe**. It can be accessed and used from any thread without requiring synchronization, as it contains no mutable fields and its instances are singletons managed by the JVM.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntity(ctx, ref) | Ref<EntityStore> | O(1) | Resolves the enum to a specific entity reference from the provided InteractionContext. This is the primary method for resolving the target of an action. Returns null if the role is not filled in the context. |
| getEntity(ctx, ref, clazz) | <T extends Entity> | O(1) | **Deprecated.** Resolves the enum to a legacy Entity object and performs a runtime type check. Use of this method is discouraged in favor of the modern reference-based API. |
| toProtocol() | com.hypixel.hytale.protocol.InteractionTarget | O(1) | Translates the server-side enum into its equivalent representation for the network protocol layer, enabling communication with the client. |

## Integration Patterns

### Standard Usage
The primary pattern is to use InteractionTarget as a selector to retrieve a specific entity from an InteractionContext. This is common in systems that process game events, apply effects, or execute scripted behaviors defined in asset files.

```java
// Assume 'effectDefinition' is loaded from a game asset file
// and effectDefinition.getTargetSelector() returns an InteractionTarget enum.

InteractionTarget selector = effectDefinition.getTargetSelector();
InteractionContext ctx = ...; // The context of the current event

// Resolve the selector to an actual entity reference
Ref<EntityStore> targetEntityRef = selector.getEntity(ctx, someDefaultRef);

if (targetEntityRef != null) {
    // Apply logic to the resolved entity
    applyHealingEffect(targetEntityRef);
}
```

### Anti-Patterns (Do NOT do this)
- **Null Context:** Passing a null InteractionContext to any `getEntity` method will result in a NullPointerException. The context is a mandatory dependency for resolution.
- **Ignoring Deprecation:** Avoid using the deprecated `getEntity(ctx, ref, clazz)` method. It relies on a legacy entity lookup path and performs an unsafe cast, which can lead to runtime ClassCastExceptions and is less performant than the reference-based approach.
- **Assuming Existence:** Do not assume `getEntity` will always return a non-null value. An interaction context may not have an OWNER or a TARGET. Always perform a null check on the returned entity reference.

## Data Pipeline
InteractionTarget is not a processing stage in a pipeline but rather a piece of configuration data that *directs* the flow of logic. It is typically deserialized at the start of a process and used as a routing key to select the correct data for subsequent stages.

> Flow:
> Game Asset (JSON/HOCON) -> Deserializer (using CODEC) -> **InteractionTarget instance** -> Interaction Logic -> Entity Resolution -> Game State Mutation

---

