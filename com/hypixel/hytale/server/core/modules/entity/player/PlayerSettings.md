---
description: Architectural reference for PlayerSettings
---

# PlayerSettings

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Data Component

## Definition
```java
// Signature
public record PlayerSettings(
   boolean showEntityMarkers,
   @Nonnull PickupLocation armorItemsPreferredPickupLocation,
   @Nonnull PickupLocation weaponAndToolItemsPreferredPickupLocation,
   @Nonnull PickupLocation usableItemsItemsPreferredPickupLocation,
   @Nonnull PickupLocation solidBlockItemsPreferredPickupLocation,
   @Nonnull PickupLocation miscItemsPreferredPickupLocation,
   PlayerCreativeSettings creativeSettings
) implements Component<EntityStore> {
```

## Architecture & Concepts
The PlayerSettings record is a pure data component within the server's Entity-Component-System (ECS) architecture. It does not contain any logic; its sole responsibility is to hold configuration state specific to a player entity. This state primarily governs client-side preferences and automated inventory behaviors, such as where different categories of items should be placed upon pickup.

As an implementation of the Component interface, it is designed to be attached to a player entity and managed by an EntityStore. Systems, such as an InventorySystem or a PlayerInteractionSystem, query this component from a player entity to make decisions. The static method getComponentType provides a runtime-stable identifier, allowing for efficient, type-safe lookups within the ECS framework.

The use of a Java record enforces immutability, a critical design choice that simplifies state management and eliminates entire classes of concurrency-related bugs.

### Lifecycle & Ownership
-   **Creation:** A PlayerSettings component is created and attached to a player entity when that entity is first initialized in the world. It is typically instantiated using the static defaults method as a template, which is then potentially overridden with data loaded from persistent storage for that specific player. The clone method facilitates the creation of new, modified instances when settings are updated.
-   **Scope:** The component's lifetime is strictly bound to the player entity it is attached to. It persists as long as the player is active in the EntityStore.
-   **Destruction:** The component is marked for garbage collection when the parent player entity is removed from the world or the server shuts down. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** **Immutable**. By virtue of being a Java record, all fields are final. Any change to a player's settings requires creating a new PlayerSettings instance and replacing the old component on the entity. This pattern ensures predictable state transitions.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, an instance of PlayerSettings can be safely read by any number of systems across multiple threads without locks or synchronization. This is a significant performance and stability advantage in a multi-threaded server environment.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component, used for ECS lookups. |
| defaults() | static PlayerSettings | O(1) | Returns a shared, static instance with default game settings. |
| clone() | PlayerSettings | O(1) | Creates a deep copy of the component, including its nested creative settings. |

## Integration Patterns

### Standard Usage
Systems should retrieve this component from a player entity via the EntityStore to read player preferences. To update settings, a system must create a new instance and replace the existing component on the entity.

```java
// A hypothetical system processing an item pickup for a player entity
Entity playerEntity = ...;
PlayerSettings settings = playerEntity.getComponent(PlayerSettings.getComponentType());

// Use the settings to determine behavior
if (settings.miscItemsPreferredPickupLocation() == PickupLocation.Hotbar) {
    // Attempt to place the item in the player's hotbar
}
```

### Anti-Patterns (Do NOT do this)
-   **Attempted Mutation:** Do not attempt to modify a PlayerSettings instance. The Java record structure makes this impossible, but developers should not build logic that assumes mutability. The correct pattern is to create a new record with the desired changes.
-   **Misusing Defaults:** Do not attach the shared instance from the static defaults method directly to a player entity. This static instance should only be used as a template for creating a new component for a new player.

## Data Pipeline
PlayerSettings acts as a data source for various game systems. Its data originates from the client and is persisted on the server.

> Flow (Settings Update):
> Client UI Change -> Network Packet (PlayerSettingsUpdate) -> Server Network Handler -> **PlayerSettingsSystem** -> Creates new **PlayerSettings** instance -> Replaces component on Player Entity

> Flow (Settings Usage):
> Player Action (e.g., Item Pickup) -> InventorySystem -> Reads **PlayerSettings** from Player Entity -> Determines Item Destination -> Updates Player Inventory Component

