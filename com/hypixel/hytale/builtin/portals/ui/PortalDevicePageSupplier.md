---
description: Architectural reference for PortalDevicePageSupplier
---

# PortalDevicePageSupplier

**Package:** com.hypixel.hytale.builtin.portals.ui
**Type:** Factory / Transient

## Definition
```java
// Signature
public class PortalDevicePageSupplier implements OpenCustomUIInteraction.CustomPageSupplier {
```

## Architecture & Concepts
The **PortalDevicePageSupplier** is a server-side factory class that functions as a critical link between player interaction and the custom UI system for portal devices. It does not manage a UI directly; instead, its sole responsibility is to determine *which* UI page, if any, should be presented to a player when they interact with a block configured as a portal device.

This class is invoked by the server's core **Interaction System** when a player triggers an **OpenCustomUIInteraction**. The supplier's logic is a decision tree that evaluates the state of the world, the target block, and the associated block entity components to make a context-aware decision.

Based on its analysis, it can:
1.  Instantiate a **PortalDeviceActivePage** if an existing, valid portal is found.
2.  Instantiate a **PortalDeviceSummonPage** if a new portal device is being created.
3.  Perform a corrective action, such as deactivating a misconfigured portal, and return null to prevent any UI from opening.
4.  Return null if the interaction context is invalid (e.g., no target block).

This pattern decouples the generic interaction system from the specific business logic of the portal feature, allowing for complex, stateful UI responses to simple world interactions.

### Lifecycle & Ownership
-   **Creation:** Instances of **PortalDevicePageSupplier** are not created directly using the new keyword. They are instantiated by the server's asset loading system via the provided static **CODEC**. This typically occurs during server startup when game assets, such as interaction configurations, are deserialized. The **PortalDeviceConfig** is injected during this deserialization process.
-   **Scope:** The object's lifetime is tied to the **OpenCustomUIInteraction** configuration that references it. It persists in memory as a stateless factory for as long as that interaction is registered with the server.
-   **Destruction:** The object is eligible for garbage collection when the server unloads or reloads its game assets, effectively discarding the old interaction configurations.

## Internal State & Concurrency
-   **State:** The class holds a single field, **config**, of type **PortalDeviceConfig**. This field is set once upon creation by the codec and should be considered immutable for the object's lifetime. The **tryCreate** method is stateless and derives all its logic from the arguments passed to it and the world state it queries.

-   **Thread Safety:** **WARNING:** This class is **not thread-safe**. The **tryCreate** method reads and writes directly to the world state (e.g., **world.setBlockInteractionState**, **chunkStore.getStore().putComponent**). It is designed to be called exclusively from the main server thread, which has synchronous access to world data. Invoking this method from any other thread will lead to race conditions, data corruption, and server instability.

## API Surface
The public contract is exclusively defined by its implementation of the **CustomPageSupplier** interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryCreate(ref, store, playerRef, context) | CustomUIPage | O(C) | Factory method. Analyzes the interaction context and world state to conditionally create and return a specific UI page instance. Returns null if no UI should be shown. Throws assertions if core components like the Player are missing. |

*Complexity is noted as O(C) or "Constant-time (world access)", as operations depend on chunk lookups which are highly optimized.*

## Integration Patterns

### Standard Usage
A developer or content creator does not invoke this class directly in Java code. Instead, it is specified within a JSON asset file that defines an **OpenCustomUIInteraction**. The server's interaction system handles the invocation.

A conceptual asset definition would look like this:
```json
// in some_interaction.json
{
  "type": "hytale:open_custom_ui",
  "pageSupplier": {
    "type": "hytale:portal_device_page_supplier",
    "Config": {
      // PortalDeviceConfig fields go here
      "onState": "on",
      "offState": "off",
      "blockStates": ["on", "off", "frame"]
    }
  }
}
```
The engine then uses the **CODEC** to instantiate **PortalDevicePageSupplier** with the provided configuration.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PortalDevicePageSupplier()`. The internal **config** field will be null, leading to a **NullPointerException** when **tryCreate** is called. Always define it within an interaction asset so the codec can construct it properly.
-   **Asynchronous Invocation:** Do not call the **tryCreate** method from a separate thread or asynchronous task. All world modifications must occur on the main server thread to prevent catastrophic state corruption.
-   **State Misconfiguration:** Ensure that all block state keys provided in the configuration (e.g., **onState**, **offState**) correspond to valid block states for the target block. The system includes checks, but misconfiguration will lead to error messages for players and prevent the portal from functioning.

## Data Pipeline
The flow of data and control for a typical interaction leading to a UI page is as follows. The **PortalDevicePageSupplier** is the central decision-making component in this flow.

> Flow:
> Player Interaction (Input) -> Server Network Layer -> Interaction System identifies target block -> **PortalDevicePageSupplier.tryCreate()** is invoked -> Class queries World State (Block Type, Components) -> Returns new **CustomUIPage** instance -> Server sends "Open UI" network message to client -> Client-side UI renders the page.

