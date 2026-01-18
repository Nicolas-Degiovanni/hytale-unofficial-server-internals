---
description: Architectural reference for CursedItems
---

# CursedItems

**Package:** com.hypixel.hytale.builtin.portals.utils
**Type:** Utility

## Definition
```java
// Signature
public final class CursedItems {
```

## Architecture & Concepts
The CursedItems class is a stateless, domain-specific utility designed for batch processing of items within an inventory. Its sole responsibility is to identify and manipulate items that possess a "cursed" status, as defined by their attached AdventureMetadata.

This class acts as a pure function library, encapsulating the business logic for two primary operations: removing the cursed status from all items in a container (uncurseAll) and completely removing all cursed items from a container (deleteAll). By centralizing this logic, it decouples game systems—such as portal transitions, zone entries, or quest completions—from the underlying implementation of the inventory and item metadata systems. It provides a clean, high-level API for what would otherwise be a verbose and error-prone iteration and modification loop.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its constructor is private to enforce its status as a static utility class.
- **Scope:** As a stateless utility, CursedItems has no lifecycle. Its static methods are available for the entire duration of the server's runtime.
- **Destruction:** Not applicable. The class is unloaded by the JVM when the server shuts down.

## Internal State & Concurrency
- **State:** CursedItems is entirely stateless. It contains no member fields and all operations are performed exclusively on the arguments provided to its static methods. The outcome of a method call depends only on the input `ItemContainer` or `Player`.

- **Thread Safety:** The methods are thread-safe by design. However, the objects they operate on, such as `ItemContainer`, are not. The caller bears the full responsibility for ensuring that no other thread is modifying the `ItemContainer` instance while a CursedItems method is executing. Failure to provide this external synchronization will likely result in `ConcurrentModificationException` or other undefined container states.

    **WARNING:** The internal use of `AtomicBoolean` in `uncurseAll` is a mechanism for safely accumulating a result from within a lambda expression. It does not make the operation on the `ItemContainer` itself atomic or thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| uncurseAll(ItemContainer) | boolean | O(N) | Iterates the container, removing the cursed flag from any item with AdventureMetadata. Returns true if at least one item was modified. |
| deleteAll(Player) | void | O(N) | A convenience method that removes all cursed items from a player's combined inventory. |
| deleteAll(ItemContainer) | void | O(N) | Iterates the container, replacing any cursed item with an empty ItemStack. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to retrieve an inventory container from a game entity, such as a Player, and pass it to the appropriate static method to perform a bulk update.

```java
// Example: Removing all curses from a player's inventory upon
// completing a purification quest.
Player player = ...;
ItemContainer inventory = player.getInventory().getCombinedEverything();

boolean itemsWereUncursed = CursedItems.uncurseAll(inventory);

if (itemsWereUncursed) {
    player.sendMessage("Your items feel lighter.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is impossible to create an instance of CursedItems via `new CursedItems()`. Do not attempt to modify the class to allow this.
- **Asynchronous Modification:** Do not modify an `ItemContainer` from another thread while a CursedItems method is operating on it. This will break the container's internal state.
- **Ignoring Return Value:** The boolean returned by `uncurseAll` is significant. It indicates whether a change was actually made, which can be critical for triggering subsequent game events or player feedback. Do not ignore it.

## Data Pipeline
CursedItems is a processing stage within a larger data flow, not an originator of data. It transforms an `ItemContainer` in-place based on a specific set of rules.

> **Uncurse Flow:**
> Game Event (e.g., Zone Entry) -> Get `Player` -> Get `ItemContainer` -> **CursedItems.uncurseAll** -> Modified `ItemContainer` -> Game Logic (checks boolean result)

> **Deletion Flow:**
> Game Event (e.g., Player Death in Hardcore) -> Get `Player` -> **CursedItems.deleteAll** -> Modified `ItemContainer` (items are now `ItemStack.EMPTY`)

