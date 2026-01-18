---
description: Architectural reference for SystemGroup
---

# SystemGroup

**Package:** com.hypixel.hytale.component
**Type:** Transient Handle

## Definition
```java
// Signature
public class SystemGroup<ECS_TYPE> implements Comparable<SystemGroup<ECS_TYPE>> {
```

## Architecture & Concepts
The SystemGroup class is a lightweight, non-owning handle that represents a collection of related systems within the Entity-Component-System (ECS) architecture. It does not contain the systems themselves; rather, it acts as a stable identifier and metadata container for a logical grouping managed by a parent ComponentRegistry.

Its primary role is to facilitate the scheduling and execution of game logic. The engine organizes systems into groups that often correspond to distinct phases of the game loop, such as *InputProcessing*, *PhysicsUpdate*, or *RenderPreparation*. The SystemGroup provides the necessary metadata—specifically an execution index and a set of dependencies—that allows an ECS scheduler to build a valid execution graph.

This class is fundamental to ensuring that systems run in the correct order. For example, a group containing physics calculations must execute before a group that renders object positions. This is enforced through the dependency mechanism.

**WARNING:** A SystemGroup is inextricably linked to the ComponentRegistry that created it. Attempting to use a handle with a different registry will result in a runtime exception.

## Lifecycle & Ownership
- **Creation:** SystemGroup instances are created exclusively by a ComponentRegistry. The constructor is package-private, which strictly prohibits direct instantiation from application-level code. A developer receives a handle by defining a group within the registry.
- **Scope:** The lifetime of a SystemGroup is tied to the configuration of its parent ComponentRegistry. It remains valid as long as the registry's system configuration is unchanged.
- **Destruction:** A SystemGroup is not destroyed in a traditional sense but is *invalidated*. If the parent ComponentRegistry is reconfigured (e.g., systems are added or removed), it will call the internal invalidate method on all existing SystemGroup handles. Subsequent use of an invalidated handle will throw an IllegalStateException. The Java garbage collector will reclaim the memory once all references to the handle are dropped.

## Internal State & Concurrency
- **State:** The object's core state (registry, index, dependencies) is immutable and established at creation. However, its validity state, represented by the internal *invalidated* flag, is mutable.
- **Thread Safety:** This class is **not thread-safe**. The internal *invalidated* flag is not volatile or atomic. The class is designed to be managed and accessed by a single thread, typically the main engine thread responsible for the ECS update cycle. Passing this handle across threads without external synchronization is a severe anti-pattern and will lead to unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRegistry() | ComponentRegistry | O(1) | Returns the parent registry that owns this group. |
| getDependencies() | Set | O(1) | Returns the set of other SystemGroups this group depends on. |
| getIndex() | int | O(1) | Returns the sorting index, used for primary execution ordering. |
| validateRegistry(registry) | void | O(1) | Throws IllegalArgumentException if the handle is used with the wrong registry. |
| validate() | void | O(1) | Throws IllegalStateException if the handle has been invalidated. |
| compareTo(other) | int | O(1) | Compares this group to another based on their index for sorting. |

## Integration Patterns

### Standard Usage
A developer typically never interacts with this class directly. Instead, they define system groups through the ComponentRegistry API and receive a SystemGroup handle in return. This handle is then used to assign new systems to that group.

```java
// A ComponentRegistry instance is assumed to exist
ComponentRegistry ecs = getEcsRegistry();

// 1. Define a new group and receive its handle
SystemGroup physicsGroup = ecs.defineSystemGroup("Physics");

// 2. Use the handle to add a system to that group
ecs.addSystem(new RigidBodySystem(), physicsGroup);

// The engine's internal scheduler will now use the metadata from
// the physicsGroup handle to correctly order the RigidBodySystem execution.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is package-private for a reason. Never attempt to create a SystemGroup using reflection or other means. It will not be registered with the engine and will be completely non-functional.
- **Using an Invalidated Handle:** Do not cache SystemGroup handles across major state changes, such as unloading a world or reloading game assets. The parent registry may invalidate all handles, and continued use will cause a crash. Always re-fetch handles from the registry after such events.
- **Cross-Registry Usage:** A handle from the server's ComponentRegistry cannot be used with the client's ComponentRegistry. The `validateRegistry` method exists to prevent this, but code should not rely on this check.

## Data Pipeline
A SystemGroup is not part of a data processing pipeline itself. Instead, it is a metadata component that *defines* the pipeline for the ECS scheduler.

> Flow:
> 1. **Engine Bootstrap**: ComponentRegistry creates a set of SystemGroup instances based on engine or game configuration.
> 2. **Scheduler Initialization**: The ECS scheduler collects all SystemGroup handles from the registry.
> 3. **Graph Construction**: The scheduler uses the `getIndex()` and `getDependencies()` from each SystemGroup to perform a topological sort, creating a final, linear execution order for all systems.
> 4. **Game Loop Tick**: The scheduler iterates through the sorted list of systems and executes them. The SystemGroup object itself is not used during this phase.

