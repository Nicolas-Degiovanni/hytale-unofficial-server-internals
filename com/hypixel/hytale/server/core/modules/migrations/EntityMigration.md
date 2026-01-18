---
description: Architectural reference for EntityMigration
---

# EntityMigration

**Package:** com.hypixel.hytale.server.core.modules.migrations
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class EntityMigration<T> implements Migration {
```

## Architecture & Concepts
The EntityMigration class is a specialized, abstract template within the server's world data versioning framework. It provides a strongly-typed contract for upgrading entity data structures stored within a WorldChunk. This system is critical for maintaining forward compatibility of world saves as the game evolves.

Unlike a generic Migration, which operates on an entire chunk, an EntityMigration is designed to target and transform only a specific class of entity. The core architectural pattern is a form of the **Strategy Pattern**, where each concrete implementation defines a specific algorithm (the migration logic) for a specific entity type.

A higher-level MigrationManager is responsible for discovering concrete EntityMigration implementations, identifying which chunks require which migrations, and orchestrating the process. This class deliberately breaks the contract of its parent interface, Migration, by disabling the generic run method. This forces developers to engage with the entity-specific migration pipeline, preventing incorrect or inefficient chunk-level operations.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses are instantiated by the server's MigrationManager during the world loading and version validation phase. The system typically uses reflection or a service registry to discover all available migration strategies.
- **Scope:** Transient and short-lived. An instance of an EntityMigration exists only for the duration of the migration process on a specific world. It is not a long-lived service.
- **Destruction:** The object is eligible for garbage collection as soon as the world migration process is complete. It holds no persistent resources that require explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state, consisting of the target entity class and an ExtraInfo supplier, is defined at construction and is effectively immutable. The object itself is stateless with respect to the migration operations it performs; its purpose is to encapsulate logic, not data.
- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking mechanisms. The controlling MigrationManager is responsible for all concurrency management. It is assumed that a single EntityMigration instance will operate on entities within a single chunk in a single-threaded context. Do not share instances across threads.

## API Surface
The public contract is minimal, focusing exclusively on the implementation of the migration logic itself.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| EntityMigration(Class, IntFunction) | constructor | O(1) | Constructs the migration strategy for a specific entity class. |
| run(WorldChunk) | final void | N/A | **WARNING:** Throws UnsupportedOperationException. This method is intentionally disabled to enforce the entity-specific migration pattern. |
| migrate(T) | abstract boolean | O(N) | The core migration logic to be implemented by subclasses. The return value should indicate if the entity was modified. |

## Integration Patterns

### Standard Usage
A developer must extend this class, providing a concrete type and implementing the migrate method. The server's migration system will then automatically discover and apply it during world upgrades.

```java
// Example: A migration to update an older Zombie entity
public class ZombieV2Migration extends EntityMigration<ZombieEntity> {

    public ZombieV2Migration() {
        // Provide the target class and a supplier for codec info
        super(ZombieEntity.class, ExtraInfo.DEFAULT_SUPPLIER);
    }

    @Override
    protected boolean migrate(ZombieEntity zombie) {
        // Logic to upgrade the entity from an older data format
        if (zombie.needsHealthUpgrade()) {
            zombie.setHealth(100);
            return true; // Return true to indicate a change was made
        }
        return false;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Calling run:** Never call the `run(WorldChunk)` method directly. It is designed to fail and is a strong signal that you are using the class outside its intended framework.
- **Generic Type Mismatch:** Do not provide a superclass like `Entity.class` to the constructor. Migrations should be as specific as possible to avoid unintended side effects on child entity classes.
- **External State Modification:** The `migrate` method should be a pure function with respect to the game state. It should only modify the entity instance passed to it. Modifying other entities or global state from within this method can lead to race conditions and non-deterministic behavior, as the order of chunk migrations is not guaranteed.

## Data Pipeline
EntityMigration acts as a data transformation step during the world loading sequence. It does not process a continuous stream of data but rather performs a one-time batch operation.

> **Flow:**
> World Load Begins -> Server Checks World Version -> **MigrationManager** Discovers and Instantiates **ConcreteEntityMigration** -> Manager Iterates Entities in a **WorldChunk** -> **migrate(entity)** is called for each matching entity -> Modified Entity Data is Persisted -> World Load Continues

