---
description: Architectural reference for InteractionTypeUtils
---

# InteractionTypeUtils

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Utility

## Definition
```java
// Note: This class is effectively final and contains only static members.
// It is not designed to be instantiated.
public class InteractionTypeUtils {
```

## Architecture & Concepts
InteractionTypeUtils is a static utility class that serves as a centralized, read-only repository for the default rules governing entity interactions. It provides the foundational logic for interaction cooldowns and mutual exclusion, acting as the server's single source of truth for baseline interaction behavior.

The primary architectural function of this class is to decouple the high-level **InteractionService** from the low-level, game-design-specific rules. Instead of hard-coding cooldowns or blocking logic within the core interaction processing pipeline, that system queries this utility class to retrieve the default parameters. This allows for clean separation of concerns and makes the default game mechanics easily auditable.

The use of performant, immutable collections like EnumSet and EnumMap is a deliberate design choice. It ensures that runtime lookups are extremely fast and that the default rules cannot be accidentally modified at runtime, guaranteeing predictable behavior across the server.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its static fields are initialized by the JVM ClassLoader when the class is first referenced by another part of the server codebase. The static initializer constructs the immutable collections.
- **Scope:** The static data within this class is global and persists for the entire lifetime of the server application.
- **Destruction:** The class and its associated static data are garbage collected by the JVM during server shutdown.

## Internal State & Concurrency
- **State:** The class contains no instance state. Its static fields, such as DEFAULT_INTERACTION_BLOCKED_BY, are **deeply immutable**. They are initialized once during class loading and are wrapped in unmodifiable collection views. This design prevents any runtime modification of the server's default interaction rules.
- **Thread Safety:** This class is inherently **thread-safe**. All data is immutable and all methods are pure functions with no side effects. It can be safely accessed by any system on any thread without requiring locks or other synchronization primitives.

## API Surface
The public API consists of immutable static fields for rule definition and static methods for querying those rules.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| STANDARD_INPUT | static final Set | O(1) | An immutable set of common, player-initiated interaction types like Primary or Use. |
| DEFAULT_INTERACTION_BLOCKED_BY | static final Map | O(1) | An immutable map defining which interaction types are mutually exclusive by default. The key is the interaction being attempted, and the value is the set of active interactions that would block it. |
| getDefaultCooldown(type) | float | O(1) | Returns the default server-wide cooldown in seconds for a given InteractionType. |
| isCollisionType(type) | boolean | O(1) | Returns true if the given InteractionType is a physics-based collision event. |

## Integration Patterns

### Standard Usage
This class is not instantiated. Its static members are accessed directly by systems that need to enforce interaction rules, typically the server's core InteractionService or systems that load entity configurations.

```java
// Example: An InteractionService checking if a new interaction is blocked.
InteractionType activeInteraction = player.getActiveInteraction();
InteractionType requestedInteraction = newInteractionEvent.getType();

// Retrieve the set of interactions that block the requested one.
Set<InteractionType> blockers = InteractionTypeUtils.DEFAULT_INTERACTION_BLOCKED_BY.get(requestedInteraction);

if (blockers != null && blockers.contains(activeInteraction)) {
    // Reject the new interaction as it is blocked by the current one.
    return;
}

// Example: An EntityConfigLoader setting a default cooldown.
float cooldown = InteractionTypeUtils.getDefaultCooldown(InteractionType.Primary);
entityConfig.setInteractionCooldown(InteractionType.Primary, cooldown);
```

### Anti-Patterns (Do NOT do this)
- **Attempting to Modify Collections:** The static collections are immutable. Any attempt to call methods like add or clear on them will result in an UnsupportedOperationException. This is a critical design feature for server stability.
- **Reflection-based Modification:** Do not use Java Reflection to bypass access modifiers and alter the static final fields. Modifying these rules at runtime would violate the system's core assumptions and lead to unpredictable, non-auditable behavior across the entire server.
- **Hard-coding Rules:** Avoid re-implementing the logic from this class elsewhere. Always refer to InteractionTypeUtils as the single source of truth for default interaction rules to maintain consistency.

## Data Pipeline
InteractionTypeUtils does not participate in a data processing pipeline. Instead, it acts as a static configuration source that other pipeline stages query for rule data.

> **Role:** Configuration Provider
>
> **Flow:**
> 1. **Class Loading:** The JVM loads InteractionTypeUtils, initializing its static, immutable rule sets.
> 2. **System Initialization:** Core engine services, such as an InteractionService or EntityConfigLoader, read from InteractionTypeUtils to establish baseline behavior.
> 3. **Runtime Query:** During gameplay, the InteractionService receives an event and queries InteractionTypeUtils to enforce default blocking and cooldown rules before applying game logic.
>
> **Example Runtime Flow:**
> Player Input -> Network Packet -> InteractionService -> **Query InteractionTypeUtils** -> Apply Game Logic

