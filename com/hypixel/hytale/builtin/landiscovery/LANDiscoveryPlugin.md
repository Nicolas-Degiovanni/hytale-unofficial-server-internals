---
description: Architectural reference for LANDiscoveryPlugin
---

# LANDiscoveryPlugin

**Package:** com.hypixel.hytale.builtin.landiscovery
**Type:** Singleton

## Definition
```java
// Signature
public class LANDiscoveryPlugin extends JavaPlugin {
```

## Architecture & Concepts
The LANDiscoveryPlugin is a core server plugin responsible for broadcasting the server's presence on the local area network (LAN). As a subclass of JavaPlugin, its lifecycle is managed entirely by the Hytale server's plugin framework. It acts as a high-level controller for the lower-level networking logic encapsulated within the LANDiscoveryThread.

Its primary responsibilities are:
1.  **Lifecycle Management:** It creates, starts, and terminates the LANDiscoveryThread in response to server state changes.
2.  **Event Handling:** It listens for the SingleplayerRequestAccessEvent to automatically enable or disable LAN broadcasting when a single-player world is opened to the network.
3.  **Command Registration:** It exposes its functionality to server administrators through the LANDiscoveryCommand.

This class serves as the integration point between the server's event system, its command system, and the specific task of network discovery.

### Lifecycle & Ownership
- **Creation:** Instantiated by the server's plugin loader during the server bootstrap sequence. The framework provides a JavaPluginInit context object required by the constructor.
- **Scope:** Singleton. A single instance, accessible via the static get method, persists for the entire server session.
- **Destruction:** The shutdown method is invoked by the plugin framework when the server is shutting down. This ensures the network broadcast thread is properly terminated and resources are released.

## Internal State & Concurrency
- **State:** The class maintains a mutable state through the lanDiscoveryThread field, which holds a reference to the active broadcasting thread or is null. The static instance field holds the singleton reference.
- **Thread Safety:** This class is **not thread-safe**. Its methods, particularly setLANDiscoveryEnabled, modify shared state without synchronization. It is designed to be operated by the main server thread through the plugin lifecycle callbacks and event bus. Unsynchronized calls from other threads can lead to race conditions and an inconsistent internal state.

**WARNING:** Accessing this plugin's methods from asynchronous tasks or other plugins running on different threads requires external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | LANDiscoveryPlugin | O(1) | Retrieves the global singleton instance. |
| setLANDiscoveryEnabled(boolean) | void | O(1) | Enables or disables the LAN discovery broadcast. This method creates or interrupts the underlying network thread. |
| isLANDiscoveryEnabled() | boolean | O(1) | Returns true if the discovery thread is currently active. |
| getLanDiscoveryThread() | LANDiscoveryThread | O(1) | Returns the underlying broadcast thread. **WARNING:** Direct manipulation of this thread is discouraged. |

## Integration Patterns

### Standard Usage
The most common interaction is indirect, driven by server events. For direct control, another plugin would retrieve the singleton instance and toggle the discovery state.

```java
// Example of another plugin controlling LAN discovery
LANDiscoveryPlugin discoveryPlugin = LANDiscoveryPlugin.get();

if (discoveryPlugin != null && !discoveryPlugin.isLANDiscoveryEnabled()) {
    // Enable broadcasting if it's currently off
    discoveryPlugin.setLANDiscoveryEnabled(true);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use new LANDiscoveryPlugin(). The plugin is instantiated and managed by the server framework. Attempting to create it manually will fail as the required JavaPluginInit context will be missing.
- **Lifecycle Mismanagement:** Do not call the protected lifecycle methods like setup, start, or shutdown directly. These are exclusively for the plugin manager. The start method contains a check that will throw an IllegalArgumentException if the thread already exists, indicating it is not designed for re-entry.
- **Unsynchronized Access:** Do not call setLANDiscoveryEnabled from multiple threads simultaneously. This can lead to multiple discovery threads being created or race conditions during shutdown.

## Data Pipeline
This component primarily acts on control signals rather than processing a continuous stream of data. Its flow is event-driven.

> Flow:
> SingleplayerRequestAccessEvent -> EventRegistry -> **LANDiscoveryPlugin** -> setLANDiscoveryEnabled() -> Creates/Interrupts LANDiscoveryThread

The subsequent data flow is handled by the managed thread:

> Flow (within LANDiscoveryThread):
> Server Metadata -> UDP Packet Assembly -> **LANDiscoveryThread** -> Socket Broadcast -> LAN

