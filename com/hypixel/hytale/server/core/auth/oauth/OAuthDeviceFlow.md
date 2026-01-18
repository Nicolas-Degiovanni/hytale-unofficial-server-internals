---
description: Architectural reference for OAuthDeviceFlow
---

# OAuthDeviceFlow

**Package:** com.hypixel.hytale.server.core.auth.oauth
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class OAuthDeviceFlow extends OAuthFlow {
```

## Architecture & Concepts
The OAuthDeviceFlow class is an abstract template that defines the contract for handling the server-side responsibilities of the OAuth 2.0 Device Authorization Grant. This flow is specifically designed for input-constrained devices, where a user authenticates on a secondary device (e.g., a web browser on a phone) to authorize the primary device (the Hytale server or client).

This class is not a complete implementation but rather a foundational component in the authentication state machine. It acts as a callback receiver. A higher-level service initiates the device flow request with an external identity provider. When the provider responds with the necessary user-facing information (user code, verification URI), the system invokes the `onFlowInfo` method on a concrete implementation of this class.

Its primary architectural role is to decouple the low-level network communication with the OAuth provider from the application-specific logic that must present the authorization details to the end-user or game client.

## Lifecycle & Ownership
- **Creation:** A concrete subclass of OAuthDeviceFlow is instantiated by a primary authentication service (e.g., an AccountManager or SessionManager) at the beginning of a specific device authentication attempt. It is not a singleton; a new instance is created for each flow.
- **Scope:** Transient. The object's lifetime is strictly bound to a single device authentication attempt. It exists only from the moment the flow is initiated until it either succeeds, fails, or times out.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection as soon as the authentication flow it represents is terminated. The parent OAuthFlow state machine is responsible for managing this lifecycle.

## Internal State & Concurrency
- **State:** As an abstract class, OAuthDeviceFlow is stateless. However, its concrete implementations are expected to be stateful, holding context relevant to the specific authentication attempt they represent.
- **Thread Safety:** **Not intrinsically thread-safe.** The `onFlowInfo` method is a callback that is almost certainly invoked on a network I/O thread. Implementations of this method **must** be thread-safe. Any interaction with shared server state or game-world objects must be properly synchronized or marshaled to the main server thread.

**WARNING:** Blocking operations within the `onFlowInfo` implementation can stall the network I/O thread, severely impacting server performance. Defer long-running tasks to a separate worker thread or scheduler.

## API Surface
The public contract consists of a single abstract method intended for implementation by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onFlowInfo(userCode, verificationUri, fullUri, expiresIn) | void | O(1) | Callback invoked by the OAuth state machine upon receiving a successful response from the identity provider. The implementation is responsible for processing these details. |

## Integration Patterns

### Standard Usage
A developer's primary interaction with this class is to extend it and provide a concrete implementation for the `onFlowInfo` callback. The system's authentication service will then manage the instantiation and invocation.

```java
// A concrete implementation to handle the flow details
public class MyHytaleDeviceFlow extends OAuthDeviceFlow {
    private final PlayerSession targetSession;

    public MyHytaleDeviceFlow(PlayerSession session) {
        this.targetSession = session;
    }

    @Override
    public void onFlowInfo(String userCode, String verificationUri, String fullUri, int expiresIn) {
        // This logic is now responsible for getting the info to the user.
        // It should not block and must be thread-safe.
        
        // Example: Send a packet to the game client
        PacketShowAuthCode packet = new PacketShowAuthCode(userCode, verificationUri, expiresIn);
        targetSession.sendPacket(packet);
    }
}

// Somewhere in the authentication service...
// The system creates an instance and passes it to the flow initiator.
OAuthFlow flow = new MyHytaleDeviceFlow(playerSession);
authProvider.initiateDeviceFlow(flow); // The provider will eventually call onFlowInfo
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call the `onFlowInfo` method directly. It is designed as a callback to be invoked exclusively by the underlying OAuth network layer or state machine. Manually calling it will desynchronize the authentication state.
- **Assuming Main Thread:** Do not assume `onFlowInfo` is executed on the main server thread. All logic within this method must be designed to execute safely on an I/O worker thread.
- **Reusing Instances:** Do not reuse an OAuthDeviceFlow instance for multiple authentication attempts. Each flow must have its own dedicated instance to prevent state corruption.

## Data Pipeline
This class is a critical junction in the device authentication data flow. It translates data from the network layer into an application-level event.

> Flow:
> Server Authentication Service -> HTTP Request to OAuth Provider -> HTTP Response (JSON) -> Network Layer JSON Deserializer -> **OAuthDeviceFlow.onFlowInfo()** -> Game Packet Encoder -> Network Packet to Client -> Client UI Update

