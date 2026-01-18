---
description: Architectural reference for SingleplayerModule
---

# SingleplayerModule

**Package:** com.hypixel.hytale.server.core.modules.singleplayer
**Type:** Singleton (Plugin)

## Definition
```java
// Signature
public class SingleplayerModule extends JavaPlugin {
```

## Architecture & Concepts
The SingleplayerModule is a core server plugin that establishes and manages the unique environment of a single-player game session. Architecturally, it acts as the primary bridge between the local game client and the embedded server process that runs alongside it. Its fundamental responsibility is to ensure the server's lifecycle is directly tethered to the client's.

This module enforces the single-player contract through several key mechanisms:

1.  **Process Lifecycle Tethering:** It actively monitors the Process ID (PID) of the parent game client. If the client process terminates unexpectedly, this module triggers an immediate and clean shutdown of the server, preventing orphaned server processes.
2.  **Access Control Delegation:** In a single-player context, the concept of a server-side whitelist or ban list is irrelevant. This module integrates with the AccessControlModule to install a ClientDelegatingProvider, effectively offloading all access control decisions to the game client.
3.  **Network Visibility Management:** It provides the core logic for transitioning a single-player world between different network access levels (e.g., Private, Friends Only). This involves direct manipulation of the server's network listeners via the ServerManager, binding and unbinding ports as required to make the server publicly discoverable or completely isolated.
4.  **Owner Identification:** It serves as the definitive source for identifying the "owner" of the session—the single player. This identity is established from startup options and used throughout the server to grant special permissions.

This module is conditionally active, with its primary functions enabled only when the server is launched with the SINGLEPLAYER flag.

### Lifecycle & Ownership
-   **Creation:** The SingleplayerModule is instantiated once by the server's plugin loader during the initial bootstrap sequence. Its constructor is invoked with a JavaPluginInit context provided by the core server.
-   **Scope:** The instance is a singleton that persists for the entire duration of the server session. It is accessible globally via the static `get()` method.
-   **Destruction:** The module is destroyed and eligible for garbage collection only when the HytaleServer itself is shut down. There are no explicit public cleanup methods; its lifecycle is managed entirely by the server's plugin framework.

## Internal State & Concurrency
-   **State:** The module maintains mutable state related to the server's network accessibility.
    -   **access:** The current, confirmed network access level of the server.
    -   **requestedAccess:** The target access level requested by the client, which may not yet be in effect. This state exists to handle the asynchronous nature of network configuration.
    -   **publicAddresses:** A list of discovered public network addresses for the server.
-   **Thread Safety:** This class is **not** inherently thread-safe. State-mutating methods such as requestServerAccess and updateAccess should be invoked from the main server thread to prevent race conditions.
    -   The `publicAddresses` list is stored in a CopyOnWriteArrayList, making read operations and iteration safe from any thread.
    -   The `checkClientPid` static method is executed on a separate, scheduled executor. This is safe as it does not interact with any instance-level state, operating only on static server options and services.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | SingleplayerModule | O(1) | Static accessor for the singleton instance. |
| requestServerAccess(access) | void | I/O Bound | Initiates a change in server network visibility. Binds/unbinds ports and sends a network packet to the client. Throws IllegalArgumentException if not in single-player mode. |
| updateAccess(access) | void | O(N) | Confirms and applies a new access level, updating internal state and notifying all connected players. |
| getAccess() | Access | O(1) | Returns the current, confirmed server access level. |
| getUuid() | UUID | O(1) | Static method. Returns the UUID of the single-player world owner. |
| isOwner(player) | boolean | O(1) | Static method. Checks if the given player reference is the world owner. |

## Integration Patterns

### Standard Usage
The module is primarily used to programmatically change the server's accessibility, for example, in response to a player command. Always retrieve the instance via the static `get()` method.

```java
// Example: Making the server private
if (Constants.SINGLEPLAYER) {
    SingleplayerModule module = SingleplayerModule.get();
    module.requestServerAccess(Access.Private);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SingleplayerModule()`. The plugin system is solely responsible for its creation. Using the constructor directly will break the singleton pattern and lead to a non-functional, detached instance.
-   **State Assumption:** Do not assume that calling `requestServerAccess(newAccess)` will synchronously update the value returned by `getAccess()`. The process is asynchronous and requires confirmation from the client. The `requestedAccess` field reflects the pending state.
-   **Multiplayer Misuse:** Do not attempt to use this module's access control features on a dedicated multiplayer server. The `requestServerAccess` method is hard-gated by the `Constants.SINGLEPLAYER` flag and will throw an exception.

## Data Pipeline
The process of changing server access involves a round-trip communication flow between the server-side module and the game client.

> **Flow: Access Change Request**
>
> External Trigger (e.g., PlayCommand) -> `SingleplayerModule.requestServerAccess()` -> `ServerManager` (Binds new network port) -> `EventBus` (Dispatches SingleplayerRequestAccessEvent) -> **`SingleplayerModule`** (Sends `RequestServerAccess` packet to client)
>
> **Flow: Access Change Confirmation**
>
> Client (Responds to request) -> Inbound Network Packet -> Server Packet Handler -> **`SingleplayerModule.updateAccess()`** -> `Universe` (Broadcasts confirmation message to players)

