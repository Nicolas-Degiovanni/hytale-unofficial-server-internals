---
description: Architectural reference for IWeightedElement
---

# IWeightedElement

**Package:** com.hypixel.hytale.common.map
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IWeightedElement {
```

## Architecture & Concepts
The IWeightedElement interface defines a fundamental contract for objects that can participate in weighted random selection algorithms. It is a core component of systems that require probabilistic outcomes, such as loot table generation, procedural content placement (e.g., biome features, ore veins), or AI decision-making.

By abstracting the concept of "weight" into a shared interface, the engine decouples the selection logic from the specific types of objects being selected. Any class, whether it represents a monster spawn, an item drop, or a decorative schematic, can be processed by a single, generic weighted selection utility so long as it implements this contract. This promotes code reuse and simplifies the design of probability-based game systems.

## Lifecycle & Ownership
As an interface, IWeightedElement itself has no lifecycle. The lifecycle (creation, scope, destruction) is entirely the responsibility of the concrete class that implements it.

- **Creation:** An object implementing IWeightedElement is created according to the needs of the system it belongs to. For example, an ItemDrop class implementing this interface might be instantiated and configured when the server loads its loot table definitions from disk.
- **Scope:** The lifetime of an implementing object is tied to its container. An element in a static loot table may be a singleton that persists for the entire server session, while a temporary, context-sensitive choice (e.g., a next-step option for an AI) might be a transient object that is garbage collected shortly after use.
- **Destruction:** Cleanup is handled by the owner of the implementing object.

## Internal State & Concurrency
- **State:** This interface is stateless. It defines a method contract but stores no data. Any state, including the value of the weight itself, is held by the implementing class. The weight may be a static, final field or a dynamically calculated value.
- **Thread Safety:** The interface itself is inherently thread-safe. However, the thread safety of any specific implementation is not guaranteed. Implementers are responsible for ensuring that the getWeight method can be called safely from multiple threads if the system requires it.

**WARNING:** If an implementation calculates weight based on mutable state, it must employ proper synchronization strategies to prevent race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWeight() | double | O(1) | Returns the probability weight of this element. Higher values increase the likelihood of selection. |

## Integration Patterns

### Standard Usage
The primary pattern is to create a collection of objects that implement IWeightedElement and pass them to a utility class that performs the weighted random selection.

```java
// Example of a concrete implementation
public class LootDrop implements IWeightedElement {
    private final double weight;
    private final Item item;

    public LootDrop(Item item, double weight) {
        this.item = item;
        this.weight = weight;
    }

    @Override
    public double getWeight() {
        return this.weight;
    }
    // ... other methods
}

// Example of a selection utility consuming the interface
List<IWeightedElement> lootTable = loadLoot(); // Returns a List<LootDrop>
IWeightedElement chosenElement = WeightedRandomSelector.select(lootTable);
```

### Anti-Patterns (Do NOT do this)
- **Non-Positive Weights:** Implementing getWeight to return a zero or negative value is a critical error. Most selection algorithms expect positive weights and will fail, enter an infinite loop, or produce undefined behavior if this contract is violated.
- **Volatile Weights:** Avoid implementing getWeight to return a value that changes frequently without a mechanism to notify the consuming system. If a collection of weighted elements is processed once to calculate a total weight, subsequent changes to individual weights will not be reflected, leading to incorrect probability distributions.

## Data Pipeline
The interface facilitates a simple but critical data flow for any probability-based system.

> Flow:
> Collection of Objects -> **IWeightedElement.getWeight()** -> Weighted Selection Algorithm -> Single Chosen Object

