---
description: Architectural reference for ModelSystems
---

# ModelSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Utility

## Definition
```java
// Signature
public class ModelSystems {
```

## Architecture & Concepts
The ModelSystems class is a non-instantiable utility class that serves as a namespace and container for a suite of related Entity Component Systems (ECS). It is not a system itself, but rather a logical grouping for all systems that manage the lifecycle and state of an entity's visual representation on the server.

These inner-class systems collectively handle:
-   **Model Initialization:** Assigning default or persistent models to entities when they are created (e.g., PlayerConnect, SetRenderedModel).
-   **State Synchronization:** Propagating changes in visual state (models, animations) to the network layer for client-side rendering (e.g., AnimationEntityTrackerUpdate).
-   **Physics Integration:** Updating physical properties like bounding boxes in response to model or state changes (e.g., ModelSpawned, UpdateBoundingBox).
-   **Cosmetic Application:** Applying skins and other cosmetic features (e.g., ApplyRandomSkin).

This class embodies a "System Grouping" pattern, improving code organization by co-locating systems that operate on the same domain of components.

---

# ModelSystems.AnimationEntityTrackerUpdate

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class AnimationEntityTrackerUpdate extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
This system is a critical bridge between the server-side animation state and the network layer. It operates on entities that are visible to players (have an EntityTrackerSystems.Visible component) and have active animations (an ActiveAnimationComponent).

Its sole responsibility is to detect when an entity's animation state has been marked as dirty for network transmission and to propagate that change to all relevant client connections. It belongs to the **EntityTrackerSystems.QUEUE_UPDATE_GROUP**, which places its execution within the entity tracking and network serialization phase of the server tick. This ensures animation updates are processed just before network packets are built and sent.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's main ECS runner during world initialization when systems are registered.
-   **Scope:** Lives for the duration of the server session. It is stateless and its instance is managed by the ECS framework.
-   **Destruction:** Decommissioned when the server world is shut down.

## Internal State & Concurrency
-   **State:** Stateless. All data is read directly from components within the ECS archetype chunks during the tick method.
-   **Thread Safety:** Conditionally parallel. The **isParallel** method allows the ECS runner to process multiple archetype chunks concurrently. This is safe because each tick invocation operates on a unique entity index and queues updates to viewer-specific lists, preventing cross-thread data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, ...) | void | O(N) | Called per entity per tick. Checks if animation state is outdated and queues network updates for N visible players. |

## Integration Patterns

### Standard Usage
This system is entirely automated. A developer triggers its behavior by modifying an entity's **ActiveAnimationComponent** and calling a method that flags its network state as outdated. The system will automatically detect this change on the next tick and handle the replication.

```java
// Somewhere in another system's logic...
ActiveAnimationComponent anim = store.getComponent(entityRef, ActiveAnimationComponent.class);
anim.playAnimation("walk"); // This internally flags the component as network-outdated
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call the tick method directly. This bypasses the ECS scheduler and its concurrency management, leading to race conditions and state corruption.
-   **Manual Packet Creation:** Do not attempt to replicate animation state manually. This system is the single source of truth for animation network updates.

## Data Pipeline
> Flow:
> ActiveAnimationComponent state changes -> **AnimationEntityTrackerUpdate** tick method runs -> Creates ComponentUpdate packet -> Queues packet on EntityViewer for each visible player -> Network layer sends packet to client.

---

# ModelSystems.ApplyRandomSkin

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class ApplyRandomSkin extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system automates the cosmetic appearance of certain entities. It is an event-driven system that triggers when a new entity is created that matches its query: having both a **ModelComponent** and an **ApplyRandomSkinPersistedComponent**.

Upon entity creation, this system interfaces with the **CosmeticsModule** to generate a random skin and attaches it to the entity as a new **PlayerSkinComponent**. This is a classic "on-spawn initialization" pattern within the ECS, ensuring that entities configured for random skins receive one immediately upon entering the world.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner during world initialization.
-   **Scope:** Persists for the server session, managed by the ECS framework.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. HolderSystem callbacks like **onEntityAdd** are executed serially by the ECS framework as entities are added to matching archetypes.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, ...) | void | O(1) | Triggered when a matching entity is created. Generates and applies a random skin. |

## Integration Patterns

### Standard Usage
A developer enables this behavior by adding the **ApplyRandomSkinPersistedComponent** to a prefab or entity definition. The system handles the rest automatically upon entity instantiation.

```java
// In a prefab definition or entity creation logic
CommandBuffer commands = ...;
Ref<EntityStore> newEntity = commands.createEntity();
commands.addComponent(newEntity, new ModelComponent(...));
commands.addComponent(newEntity, new ApplyRandomSkinPersistedComponent()); // This will trigger the system
```

### Anti-Patterns (Do NOT do this)
-   **Manual Skin Application:** If an entity has the **ApplyRandomSkinPersistedComponent**, do not also manually add a **PlayerSkinComponent**. This can lead to race conditions or overwriting the desired random skin.

## Data Pipeline
> Flow:
> Entity with **ApplyRandomSkinPersistedComponent** is created -> **ApplyRandomSkin** onEntityAdd is triggered -> Calls **CosmeticsModule.generateRandomSkin** -> A new **PlayerSkinComponent** is added to the entity.

---

# ModelSystems.AssignNetworkIdToProps

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class AssignNetworkIdToProps extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This is an initialization system responsible for network identity assignment. Its query specifically targets entities that have a **PropComponent** but do *not* yet have a **NetworkId** component.

When a new prop entity is created, this system triggers and assigns it a unique network ID from the world's central pool. This ensures that all prop entities are uniquely identifiable by clients for purposes of rendering and interaction, separating them from static world geometry.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner during world initialization.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless. It retrieves the next network ID from an external source (**store.getExternalData()**).
-   **Thread Safety:** Not thread-safe. Entity addition events are processed serially. The call to **takeNextNetworkId** must be atomic to prevent ID collisions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, ...) | void | O(1) | Triggered when a prop entity without a NetworkId is created. Assigns a new NetworkId. |

## Integration Patterns

### Standard Usage
This system is fully automated. Creating an entity with a **PropComponent** is sufficient to trigger it.

```java
// Creating a prop entity will automatically trigger this system
CommandBuffer commands = ...;
Ref<EntityStore> prop = commands.createEntity();
commands.addComponent(prop, new PropComponent());
// The system will now add a NetworkId component
```

### Anti-Patterns (Do NOT do this)
-   **Pre-assigning NetworkId:** Do not manually add a **NetworkId** component to a prop prefab. This system is designed to be the sole authority for assigning these IDs at runtime to prevent collisions.

---

# ModelSystems.ModelChange

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class ModelChange extends RefChangeSystem<EntityStore, ModelComponent> {
```

## Architecture & Concepts
This is a reactive system that synchronizes an entity's runtime model with its persistent state. It listens for any changes (additions, modifications, removals) to the **ModelComponent** on entities that also have a **PersistentModel** component.

Its purpose is to ensure that if an entity's visual model is changed during gameplay (e.g., through an item effect or transformation), that change is reflected in the **PersistentModel** component. This guarantees the change will be saved correctly if the entity is unloaded and later reloaded.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. Component change events are processed serially by the ECS framework.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentSet(...) | void | O(1) | Triggered when a ModelComponent is changed. Updates the associated PersistentModel. |
| onComponentRemoved(...) | void | O(1) | Triggered when a ModelComponent is removed. Removes the associated PersistentModel. |

## Integration Patterns

### Standard Usage
This system works in the background. Any game logic that changes an entity's **ModelComponent** on a persistent entity will automatically trigger this system to update the persistent state.

```java
// Change the model on an entity that has a PersistentModel component
commandBuffer.setComponent(entityRef, new ModelComponent(newModel));
// ModelChange system will now automatically update the PersistentModel component
```

### Anti-Patterns (Do NOT do this)
-   **Manual Synchronization:** Do not manually update both the **ModelComponent** and **PersistentModel** component. This is redundant and can lead to race conditions. Rely on this system to perform the synchronization.

## Data Pipeline
> Flow:
> **ModelComponent** is set or removed on a persistent entity -> **ModelChange** system callback is triggered -> The system updates or removes the corresponding **PersistentModel** component.

---

# ModelSystems.ModelSpawned

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class ModelSpawned extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This is an initialization system that configures an entity's physical properties based on its visual model. It triggers when an entity with a **ModelComponent** is added to the world.

The system reads the model's predefined bounding box and detail boxes and applies them to the entity's **BoundingBox** component. This is a crucial step for integrating a new entity into the physics and collision system.

Crucially, it declares a dependency to run *after* **SetRenderedModel**. This ensures that if an entity is being loaded from persistence, its model is first set from the **PersistentModel** component before this system attempts to read the model's bounding box.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. Entity addition events are processed serially.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, ...) | void | O(1) | Triggered when an entity with a ModelComponent is created. Configures its BoundingBox. |

## Integration Patterns

### Standard Usage
This system is automated. Adding a **ModelComponent** to an entity is sufficient to trigger it.

```java
// This action will trigger the ModelSpawned system
commandBuffer.addComponent(entityRef, new ModelComponent(someModel));
// The entity's BoundingBox component will now be configured based on someModel
```

### Anti-Patterns (Do NOT do this)
-   **Manual Bounding Box Setup:** Avoid manually setting the bounding box from a model's data. This system is the designated authority for this task, ensuring it happens at the correct point in the entity initialization sequence.

## Data Pipeline
> Flow:
> Entity with **ModelComponent** is created -> **ModelSpawned** onEntityAdd is triggered -> Reads bounding box from **Model** asset -> Sets data on the entity's **BoundingBox** component.

---

# ModelSystems.PlayerConnect

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class PlayerConnect extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system is responsible for assigning a visual model to a player entity when it first joins the server. It targets entities that have a **Player** component but do *not* yet have a **ModelComponent**.

It reads the player's configured model preset from their profile data. If a valid preset is found, it loads the corresponding model asset and attaches it as a **ModelComponent**. If not, it falls back to a default "Player" model.

This system is ordered to run *before* **ModelSpawned**. This dependency is critical: **PlayerConnect** first adds the **ModelComponent**, and then **ModelSpawned** runs to configure the player's bounding box based on that newly added model. This creates a reliable, ordered initialization chain for player entities.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. Player connection events are processed serially.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, ...) | void | O(1) | Triggered when a Player entity is created. Loads their model and adds a ModelComponent. |

## Integration Patterns

### Standard Usage
This system is fully automated and is a core part of the player connection flow.

### Anti-Patterns (Do NOT do this)
-   **Pre-adding ModelComponent:** Do not add a **ModelComponent** to a player entity before this system runs. Its query relies on the absence of this component to trigger correctly.

## Data Pipeline
> Flow:
> Player entity is created -> **PlayerConnect** onEntityAdd is triggered -> Reads player config data -> Loads **ModelAsset** from asset store -> Creates and adds **ModelComponent** to the player entity.

---

# ModelSystems.PlayerUpdateMovementManager

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class PlayerUpdateMovementManager extends RefChangeSystem<EntityStore, ModelComponent> {
```

## Architecture & Concepts
This is a reactive system that ensures a player's movement controller is correctly configured whenever their model changes. It listens for any changes to the **ModelComponent** on entities that also have a **Player** component.

When a player's model is added or changed, it can affect physical properties like size and collision shape. This system responds to that change by calling **resetDefaultsAndUpdate** on the player's **MovementManager** component. This forces the movement controller to re-evaluate its parameters based on the new model, ensuring movement logic remains consistent with the player's physical representation.

It is ordered to run *after* **UpdateBoundingBox**, ensuring that the bounding box is updated first before the movement controller re-initializes.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. Component change events are processed serially.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(...) | void | O(1) | Triggered when a ModelComponent is added to a player. Resets the MovementManager. |
| onComponentSet(...) | void | O(1) | Triggered when a ModelComponent is changed on a player. Resets the MovementManager. |

## Integration Patterns

### Standard Usage
This system is automated. Any game logic that changes a player's model will trigger it.

```java
// e.g., A shapeshifting ability changes the player's model
commandBuffer.setComponent(playerRef, new ModelComponent(wolfModel));
// PlayerUpdateMovementManager will now automatically reconfigure the MovementManager
```

## Data Pipeline
> Flow:
> Player's **ModelComponent** is changed -> **PlayerUpdateMovementManager** callback is triggered -> Calls **MovementManager.resetDefaultsAndUpdate** -> Player's movement physics are updated for the new model.

---

# ModelSystems.SetRenderedModel

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class SetRenderedModel extends HolderSystem<EntityStore> {
```

## Architecture & Concepts
This system is a key part of the entity loading process. It bridges the gap between an entity's persistent state and its active, in-world state. It targets entities that have been loaded with a **PersistentModel** component but do not yet have an active **ModelComponent**.

When such an entity is added to the world (e.g., loaded from disk), this system triggers. It reads the model reference from the **PersistentModel** component, resolves it to an actual **Model** object, and adds it to the entity as a new **ModelComponent**. This effectively "hydrates" the entity, making it renderable and physically present in the current game session.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. Entity addition events are processed serially.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd(holder, ...) | void | O(1) | Triggered when a persistent entity is loaded. Creates a ModelComponent from the PersistentModel. |

## Integration Patterns

### Standard Usage
This system is a core, automated part of the entity persistence pipeline. Developers do not interact with it directly.

## Data Pipeline
> Flow:
> Entity is loaded from storage with **PersistentModel** -> **SetRenderedModel** onEntityAdd is triggered -> Resolves model reference from **PersistentModel** -> Creates and adds a new **ModelComponent** to the entity.

---

# ModelSystems.UpdateBoundingBox

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class UpdateBoundingBox extends RefChangeSystem<EntityStore, ModelComponent> {
```

## Architecture & Concepts
This is a reactive system that keeps an entity's physical bounding box synchronized with its visual model. It listens for any changes to the **ModelComponent** on entities that also have a **BoundingBox** component.

Whenever an entity's model is added or changed, this system recalculates the bounding box based on the new model's data and the entity's current movement state (e.g., crouching). This ensures that the entity's collision shape always matches its visual representation.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Not thread-safe. Component change events are processed serially.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentAdded(...) | void | O(1) | Triggered when a ModelComponent is added. Updates the BoundingBox. |
| onComponentSet(...) | void | O(1) | Triggered when a ModelComponent is changed. Updates the BoundingBox. |
| onComponentRemoved(...) | void | O(1) | Triggered when a ModelComponent is removed. Resets the BoundingBox to an empty box. |

## Integration Patterns

### Standard Usage
This system is automated. Changing an entity's model will automatically trigger the bounding box update.

```java
// This will cause the UpdateBoundingBox system to fire
commandBuffer.setComponent(entityRef, new ModelComponent(largeModel));
```

## Data Pipeline
> Flow:
> **ModelComponent** is changed -> **UpdateBoundingBox** callback is triggered -> Reads new model data and current **MovementStatesComponent** -> Calculates new bounding box -> Updates the **BoundingBox** component.

---

# ModelSystems.UpdateCrouchingBoundingBox

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** Transient

## Definition
```java
// Signature
public static class UpdateCrouchingBoundingBox extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
This is an optimization system that updates an entity's bounding box specifically in response to changes in its crouching state. Unlike the reactive **UpdateBoundingBox** system, this is a ticking system that runs every frame on entities that can move and have a model.

It performs a check to see if the entity's crouching state has changed since the last network update. If, and only if, the state has changed, it re-calculates the bounding box using the model's specific crouching dimensions. This avoids expensive recalculations on every single tick for every entity, only performing the work when the state actually changes.

It is ordered to run *before* **MovementStatesSystems.TickingSystem**, ensuring the physical bounding box is updated before the main movement and physics logic processes it for the current tick.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the ECS runner.
-   **Scope:** Persists for the server session.
-   **Destruction:** Decommissioned on world shutdown.

## Internal State & Concurrency
-   **State:** Stateless.
-   **Thread Safety:** Conditionally parallel. The **isParallel** method allows the ECS runner to process multiple archetype chunks concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, ...) | void | O(1) | Called per entity per tick. Checks for crouching state changes and updates the BoundingBox if necessary. |

## Integration Patterns

### Standard Usage
This system is automated. Changing an entity's crouching state within its **MovementStatesComponent** will be detected by this system on the next tick.

## Data Pipeline
> Flow:
> Crouching state in **MovementStatesComponent** changes -> **UpdateCrouchingBoundingBox** tick method runs -> Detects discrepancy between current and sent state -> Recalculates bounding box using model's crouching data -> Updates the **BoundingBox** component.

