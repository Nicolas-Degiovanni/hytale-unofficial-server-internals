---
description: Architectural reference for RoleUtils
---

# RoleUtils

**Package:** com.hypixel.hytale.server.npc.role
**Type:** Utility

## Definition
```java
// Signature
public class RoleUtils {
```

## Architecture & Concepts
RoleUtils is a stateless utility class that provides a high-level, declarative API for manipulating the inventory of an NPCEntity. It serves as a facade over the more granular InventoryHelper, abstracting away the complexities of slot management and item usage mechanics.

The primary design goal of this class is to decouple the *definition* of an NPC's role (e.g., a "Guard" role that requires specific armor and a weapon) from the low-level *implementation* of equipping those items. By providing coarse-grained methods like setArmor and setHotbarItems, it allows role-defining logic to remain clean and focused on configuration rather than inventory mechanics.

This class also centralizes logging for failed equipment operations, ensuring consistent and informative warnings are dispatched through the NPCPlugin's logger when an item or armor set cannot be equipped.

### Lifecycle & Ownership
- **Creation:** As a static utility class, RoleUtils is never instantiated. Its bytecode is loaded by the JVM ClassLoader upon first access to any of its static members.
- **Scope:** Application-level. The class and its methods are available for the entire lifetime of the server process.
- **Destruction:** The class is unloaded by the JVM during application shutdown. There is no instance-level state to manage or clean up.

## Internal State & Concurrency
- **State:** RoleUtils is completely stateless. It contains no member variables and does not cache any data between calls. Its behavior is determined exclusively by the arguments provided to its static methods.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the operations it performs are **not atomic**. The methods mutate the state of the passed NPCEntity and its associated Inventory.

**WARNING:** Calling methods from this class on the same NPCEntity instance from multiple threads without external synchronization will lead to severe race conditions and data corruption within the NPC's inventory. All interactions with a specific NPC's inventory via this utility must be synchronized by the calling context.

## API Surface
The public API consists entirely of static methods designed for direct, procedural invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setHotbarItems(npc, items) | void | O(N) | Equips an array of items to the NPC's hotbar. N is the number of items. |
| setOffHandItems(npc, items) | void | O(N) | Equips an array of items to the NPC's off-hand slots. N is the number of items. |
| setItemInHand(npc, item) | void | O(1) | Sets the NPC's currently held item. Logs a warning on failure. |
| setArmor(npc, armor) | void | O(1) | Equips a full set of armor. Logs a warning on failure. |

## Integration Patterns

### Standard Usage
RoleUtils methods should be called directly from server-side logic responsible for initializing or updating an NPC's state, typically after the NPC is spawned or its role is changed.

```java
// Example: Equipping a newly spawned guard NPC
NPCEntity guardNpc = spawnGuard();
String[] guardHotbar = {"hytale:iron_sword", "hytale:shield"};
String guardArmor = "hytale:iron_armor_set";

// Use the utility class to configure the NPC's equipment
RoleUtils.setHotbarItems(guardNpc, guardHotbar);
RoleUtils.setArmor(guardNpc, guardArmor);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new RoleUtils()`. This is a utility class and is not designed to be instantiated. All methods are static.
- **Concurrent Modification:** Do not call RoleUtils methods on the same NPCEntity from different threads simultaneously. This will corrupt the NPC's inventory state. If multi-threaded access is required, you must implement your own locking mechanism around the NPCEntity object.

## Data Pipeline
RoleUtils acts as a command-and-control point in the data flow, translating high-level configuration into low-level state changes. It does not transform or pass through data in a traditional pipeline.

> Flow:
> NPC Role Definition Logic -> **RoleUtils.setArmor(npc, "iron_set")** -> InventoryHelper.useArmor(...) -> NPCEntity.Inventory Mutation

