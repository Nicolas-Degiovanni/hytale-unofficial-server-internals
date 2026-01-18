---
description: Architectural reference for BallisticDataProvider
---

# BallisticDataProvider

**Package:** com.hypixel.hytale.server.core.modules.projectile.config
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface BallisticDataProvider {
```

## Architecture & Concepts
The BallisticDataProvider interface establishes a formal contract for any game entity or component that can provide projectile physics parameters. It serves as a critical decoupling mechanism, allowing the server's physics and projectile simulation systems to operate on a wide variety of objects without needing to know their concrete types.

By abstracting the source of ballistic information, this interface enables a data-driven design. An entity's projectile behavior—such as gravity influence, air resistance, and terminal velocity—is not hardcoded into its class logic. Instead, systems query for this interface to retrieve a BallisticData object, which contains the necessary simulation parameters. This pattern is fundamental to Hytale's component-based entity system, promoting flexibility and content modularity.

## Lifecycle & Ownership
As an interface, BallisticDataProvider has no lifecycle of its own. However, it imposes strict lifecycle expectations on its implementers.

- **Creation:** An object implementing this interface is typically a component attached to a game entity. It is created and configured during the entity's instantiation, often from prefab or configuration data.
- **Scope:** The implementing object's lifecycle is tied directly to its parent entity. It exists as long as the entity is active in the world.
- **Destruction:** The implementing object is destroyed and garbage collected when its parent entity is removed from the game world. Consumers of this interface must not retain references to the returned BallisticData object beyond the immediate scope of their operation, as the provider may become invalid.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any class that implements BallisticDataProvider is expected to act as a provider of state—specifically, the BallisticData configuration. The returned BallisticData object should be treated as **immutable** by consumers. Modifying it may lead to undefined behavior in the physics simulation.
- **Thread Safety:** Implementations of this interface are not guaranteed to be thread-safe. It is designed to be called from the main server game loop thread. Accessing it from worker threads or asynchronous tasks without explicit synchronization is a severe anti-pattern and will lead to race conditions or inconsistent physics behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBallisticData() | BallisticData | O(1) | Retrieves the ballistic configuration for the object. Returns null if no data is available. |

**WARNING:** The return value is explicitly marked as Nullable. Consumers **must** perform a null check before attempting to use the returned BallisticData. Failure to do so is a common source of NullPointerExceptions in the projectile system.

## Integration Patterns

### Standard Usage
The standard pattern involves a system (e.g., ProjectileSimulationSystem) retrieving a component that implements this interface from an entity and then using the provided data to configure a physics step.

```java
// A system retrieves an entity and checks for the provider
BallisticDataProvider provider = entity.getComponent(BallisticDataProvider.class);

if (provider != null) {
    BallisticData data = provider.getBallisticData();
    if (data != null) {
        // Use the data to influence the projectile's trajectory
        applyGravity(projectile, data.getGravityModifier());
        applyDrag(projectile, data.getLinearDrag());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Non-Null:** Never call getBallisticData without checking for a null return. Not all entities with this provider will have valid data at all times.
- **Caching the Result:** Do not cache the returned BallisticData object across ticks. The underlying data may be dynamic and could change based on game state (e.g., an entity entering water). Always query the provider for the most current data.
- **Cross-Thread Access:** Do not call getBallisticData from a thread other than the main server thread responsible for the entity's updates.

## Data Pipeline
This interface acts as a data source, feeding configuration into the server's core physics simulation pipeline for projectile entities.

> Flow:
> Entity Configuration (JSON/Prefab) -> Entity Component (implements **BallisticDataProvider**) -> ProjectileSimulationSystem -> Physics State Update -> Network Sync to Client

