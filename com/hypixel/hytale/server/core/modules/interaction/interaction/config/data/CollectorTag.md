---
description: Architectural reference for CollectorTag
---

# CollectorTag

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.data
**Type:** Utility

## Definition
```java
// Signature
public interface CollectorTag {
   CollectorTag ROOT = new CollectorTag() {};
}
```

## Architecture & Concepts
CollectorTag is a **marker interface** used within the server's interaction system. Its primary architectural purpose is to provide a compile-time type for categorizing and identifying different groups of interaction collectors or their configurations. It does not define any methods and is not intended to be implemented with complex logic.

Classes that implement CollectorTag are effectively "tagging" themselves as a specific type of interaction category. This pattern allows the system to use these tags as unique, type-safe keys for retrieving or grouping related behaviors, often within collections like a Map.

The static constant **ROOT** represents a global, default, or top-level tag. It serves as a foundational identifier, a common fallback, or the root node in a potential hierarchy of interaction configurations.

## Lifecycle & Ownership
-   **Creation:** As an interface, CollectorTag itself is not instantiated. The provided ROOT constant is a singleton, instantiated once by the JVM during class loading. Concrete implementations are typically defined as static final constants within their own dedicated classes.
-   **Scope:** The ROOT constant is global and persists for the entire lifetime of the server application. The scope of other custom tags is similarly static and application-wide.
-   **Destruction:** The ROOT constant is only eligible for garbage collection when its class loader is unloaded, which typically occurs only at application shutdown.

## Internal State & Concurrency
-   **State:** This interface is stateless. The ROOT constant is an immutable singleton instance of an anonymous class.
-   **Thread Safety:** The interface and its ROOT constant are inherently thread-safe. All implementations of CollectorTag **must** be immutable and thread-safe, as they are used as shared identifiers across different systems and threads. Modifying the state of a tag post-creation can lead to severe and difficult-to-diagnose concurrency issues, especially if used as a key in a collection.

## API Surface
The public contract consists of a single, globally accessible constant.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ROOT | CollectorTag | O(1) | A static, globally accessible root tag. Serves as a default or top-level identifier in the interaction system. |

## Integration Patterns

### Standard Usage
CollectorTag is used to create unique, type-safe identifiers for different interaction categories. Developers should define their own tags as immutable, singleton-like classes. These tags are then used as keys to register and query for specific interaction configurations or handlers from a central registry.

```java
// 1. Define a new, specific tag
public final class PlayerActionTag implements CollectorTag {
    public static final PlayerActionTag INSTANCE = new PlayerActionTag();
    private PlayerActionTag() {} // Enforce singleton
}

// 2. Use the tag to register or retrieve data
InteractionRegistry registry = server.getInteractionRegistry();
InteractionConfig playerConfig = new InteractionConfig(...);

// Registering with the custom tag
registry.register(PlayerActionTag.INSTANCE, playerConfig);

// Retrieving with the default ROOT tag
InteractionConfig defaultConfig = registry.get(CollectorTag.ROOT);
```

### Anti-Patterns (Do NOT do this)
-   **Mutable Implementations:** Never create an implementation of CollectorTag with mutable fields. Tags are used as keys in hash-based collections; mutating their state after insertion will break the contract of HashMap or HashSet, leading to memory leaks or lost data.
-   **Excessive Anonymous Instantiation:** Avoid creating new anonymous instances (`new CollectorTag() {}`) on the fly. This defeats the purpose of having stable, well-known identifiers and can lead to unnecessary object allocation. The ROOT constant is a deliberate exception. Always prefer defining a named, final, singleton class for each tag.

## Data Pipeline
CollectorTag does not process data directly. Instead, it acts as a routing key or filter within the configuration and event data pipelines. It directs which set of rules or logic should be applied to an interaction.

> **Configuration Flow:**
> Server Configuration Files -> Configuration Parser -> **CollectorTag** (as key) -> Interaction Registry -> Game Systems
>
> **Event Flow:**
> Player Interaction Event -> Event Dispatcher -> Query Registry using **CollectorTag** -> Execute Correct Interaction Logic

