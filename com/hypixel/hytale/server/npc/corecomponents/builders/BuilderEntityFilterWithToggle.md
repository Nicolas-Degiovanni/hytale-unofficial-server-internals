---
description: Architectural reference for BuilderEntityFilterWithToggle
---

# BuilderEntityFilterWithToggle

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Component Builder (Abstract)

## Definition
```java
// Signature
public abstract class BuilderEntityFilterWithToggle extends BuilderBase<IEntityFilter> {
```

## Architecture & Concepts

BuilderEntityFilterWithToggle is an abstract base class that serves as a foundational component within the server's data-driven NPC system. It is not a concrete filter itself, but rather a template for creating entity filters that require a common, configurable on/off switch.

Its primary architectural role is to enforce a standard for toggleable components and reduce boilerplate code. By extending this class, developers can create complex entity filters while inheriting a standardized mechanism for enabling or disabling them via JSON configuration. This class acts as a bridge between the asset configuration layer (JSON files) and the runtime entity filtering logic.

The core design relies on two key components:
1.  **Builder Pattern:** This class is part of a larger factory and builder system, indicated by its extension of BuilderBase. Builders are responsible for parsing a configuration format (JSON) and producing a runtime object, in this case an object implementing the IEntityFilter interface.
2.  **Dynamic Configuration:** The use of a BooleanHolder and an ExecutionContext allows the enabled state to be resolved dynamically at runtime. This means a filter's activation is not limited to a static true/false value in a file; it can depend on game state, world variables, or other contextual information.

## Lifecycle & Ownership

-   **Creation:** Instances of concrete subclasses are not instantiated directly via game logic. They are discovered and instantiated by a central Builder Registry during server initialization or when NPC asset packs are loaded. The registry uses the `category()` method to associate the builder with the IEntityFilter type.
-   **Scope:** A single builder instance is created for each concrete filter type and persists for the entire server session. These builders are stateless templates used to configure and create filter instances.
-   **Destruction:** The builder instances are garbage collected when the server shuts down or during a full reload of the asset and builder registries.

## Internal State & Concurrency

-   **State:** The primary state is the `enabled` field, a BooleanHolder. This state is configured once from a JSON asset during the `readCommonConfig` call. After this initial parsing, the builder's configuration is effectively immutable for its lifetime.
-   **Thread Safety:** This class is not thread-safe during its configuration phase. The `readCommonConfig` method should only be called from the main asset loading thread. After configuration, methods like `isEnabled` are safe for concurrent access, provided the underlying BooleanHolder implementation is also thread-safe.

**Warning:** All state modification must be completed during the single-threaded asset loading phase to prevent race conditions at runtime.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(JsonElement) | Builder<IEntityFilter> | O(N) | Parses the "Enabled" property from a JSON object. N is the number of keys in the object. Throws exceptions on malformed JSON. |
| category() | Class<IEntityFilter> | O(1) | Returns the IEntityFilter class token, used for registration with the builder system. |
| isEnabled(ExecutionContext) | boolean | O(1) | Evaluates the configured BooleanHolder against the provided context to determine if the filter is active. |

## Integration Patterns

### Standard Usage

This class must be extended. A developer would create a new filter builder and ensure the parent's configuration method is called.

```java
// In a new class, e.g., BuilderFilterIsHostile.java
public class BuilderFilterIsHostile extends BuilderEntityFilterWithToggle {

    @Nonnull
    @Override
    public Builder<IEntityFilter> readCommonConfig(@Nonnull JsonElement data) {
        // CRITICAL: Must call super to process the "Enabled" field.
        super.readCommonConfig(data);

        // ... read other custom properties for this filter ...

        return this;
    }

    // ... implement other abstract methods to build the actual filter ...
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** It is impossible to instantiate this class with `new BuilderEntityFilterWithToggle()` because it is abstract. Do not attempt to work around this.
-   **Omitting Super Call:** Forgetting to call `super.readCommonConfig(data)` in a subclass will silently disable the "Enabled" toggle functionality. The filter will always use the default value (true), ignoring the JSON configuration.
-   **State Modification After Load:** Do not expose methods that modify the `enabled` holder after the initial asset loading phase. This breaks the "configure-once, use-many" pattern and can lead to unpredictable behavior in a multithreaded environment.

## Data Pipeline

The flow of data from configuration to runtime execution follows a clear path through the engine.

> Flow:
> NPC Asset JSON File -> Asset Loading Service -> **BuilderEntityFilterWithToggle** (via subclass) -> AI Behavior Tree -> `isEnabled(context)` Check -> Entity Filter Logic

