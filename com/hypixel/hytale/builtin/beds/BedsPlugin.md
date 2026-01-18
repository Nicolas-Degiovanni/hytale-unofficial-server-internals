---
description: Architectural reference for BedsPlugin
---

# BedsPlugin

**Package:** com.hypixel.hytale.builtin.beds
**Type:** Singleton (Plugin-Managed)

## Definition
```java
// Signature
public class BedsPlugin extends JavaPlugin {
```

## Architecture & Concepts
The BedsPlugin class is a foundational module responsible for bootstrapping the entire "sleeping in beds" gameplay feature. It operates as a self-contained plugin, discovered and loaded by the server's core plugin system at startup.

Its primary architectural role is not to implement game logic directly, but to act as a registration authority. It integrates the sleeping mechanics into the server's core Entity-Component-System (ECS) engine by performing the following key actions during its setup phase:

1.  **Component Registration:** It defines and registers the data structures that hold sleep-related state, such as PlayerSomnolence and SleepTracker. These components are attached to player entities.
2.  **Resource Registration:** It registers WorldSomnolence, a global, world-specific resource that tracks the overall sleep state of all players in a given world.
3.  **System Registration:** It registers all the systems that contain the actual sleeping logic, such as EnterBedSystem and UpdateWorldSlumberSystem. The server's main game loop is then responsible for executing these systems each tick.
4.  **Interaction Registration:** It registers the custom BedInteraction type, which links the physical act of a player right-clicking a bed block to the underlying ECS-based sleeping systems.

By centralizing these registrations, the BedsPlugin ensures that all components of the sleeping feature are initialized correctly and in the proper sequence, effectively "activating" the feature within the game world.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's central PluginLoader during the server bootstrap sequence. The constructor receives a JavaPluginInit context object, which provides access to essential server registries. Direct instantiation by developers is strictly forbidden.
-   **Scope:** The BedsPlugin instance is a session-scoped singleton. It is created once when the server starts and persists until the server shuts down.
-   **Destruction:** The object is decommissioned and becomes eligible for garbage collection only when the server is shutting down or if the plugin is explicitly unloaded by an administrator.

## Internal State & Concurrency
-   **State:** The internal state of a BedsPlugin instance consists of cached references to the ComponentType and ResourceType objects it registers (e.g., playerSomnolenceComponentType). This state is populated once during the setup method and is immutable for the remainder of the object's lifecycle. **Crucially, this class does not hold dynamic game state**; all player and world sleep data is managed within the ECS components and resources it registers.
-   **Thread Safety:** This class is thread-safe under normal operating conditions. The setup method is invoked synchronously by the main server thread during initialization. The public getter methods only return immutable references, making them safe to call from any thread. The static getInstance method is also safe, but consumers must be aware of the lifecycle.

    **Warning:** Calling BedsPlugin.getInstance() before the server's plugin loading phase is complete will result in a NullPointerException. Systems that depend on this plugin must declare a dependency to ensure correct initialization order.

## API Surface
The public API is minimal, primarily exposing the registered ECS types for use by other systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInstance() | BedsPlugin | O(1) | Returns the static, singleton instance of the plugin. |
| getPlayerSomnolenceComponentType() | ComponentType | O(1) | Returns the registered type handle for the PlayerSomnolence component. |
| getSleepTrackerComponentType() | ComponentType | O(1) | Returns the registered type handle for the SleepTracker component. |
| getWorldSomnolenceResourceType() | ResourceType | O(1) | Returns the registered type handle for the WorldSomnolence resource. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Its primary integration is with the server's plugin loader. However, other systems that need to interact with sleep-related components may retrieve the component types from the static instance.

```java
// Example from a hypothetical system that needs to check for sleepiness
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

// ... inside another system's update method ...
ComponentType<EntityStore, PlayerSomnolence> somnolenceType;
somnolenceType = BedsPlugin.getInstance().getPlayerSomnolenceComponentType();

// Use the component type to query the ECS world
for (Entity e : world.query(somnolenceType)) {
    // Logic for sleepy players
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new BedsPlugin()`. The server's plugin lifecycle manager is the sole owner of this process. Doing so will result in a broken, non-functional object that is not registered with the engine.
-   **Manual Setup:** Never call the `setup()` method directly. This method is designed to be called once by the plugin loader. Calling it manually will throw exceptions and corrupt the server's ECS registries.
-   **Early Access:** Do not attempt to access `BedsPlugin.getInstance()` from a constructor or initializer of another plugin that may load before this one. Declare an explicit plugin dependency to guarantee safe access.

## Data Pipeline
The BedsPlugin itself is not part of a data processing pipeline. Rather, it *constructs* the pipeline by registering the systems that handle the data flow for the sleeping feature. The flow for a player initiating sleep is as follows:

> Flow:
> Player Input (Use Bed) -> Server Interaction Handler -> **BedInteraction** (Registered by BedsPlugin) -> ECS Event Fired -> **EnterBedSystem** (Registered by BedsPlugin) -> Attaches PlayerSomnolence Component -> **UpdateWorldSlumberSystem** (Registered by BedsPlugin) -> Reads all PlayerSomnolence components -> Updates WorldSomnolence Resource -> Time Skips / Weather Clears

