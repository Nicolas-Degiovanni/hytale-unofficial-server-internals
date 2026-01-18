---
description: Architectural reference for OAuthBrowserFlow
---

# OAuthBrowserFlow

**Package:** com.hypixel.hytale.server.core.auth.oauth
**Type:** Utility / Framework Class

## Definition
```java
// Signature
public abstract class OAuthBrowserFlow extends OAuthFlow {
```

## Architecture & Concepts
The OAuthBrowserFlow class is an abstract base class that defines the contract for OAuth 2.0 authentication flows requiring user interaction via an external web browser. It represents a critical state in the server-side authentication sequence where the server must provide the client with a URL to open, allowing the user to grant application permissions with an identity provider (e.g., Microsoft, Google).

This class acts as a specialized state machine handler within the broader authentication framework. It decouples the core authentication service from the specifics of how a browser-based challenge is initiated. Concrete implementations of this class are responsible for handling provider-specific details, but they all adhere to the central pattern of receiving flow information (typically a URL) and propagating it to the user.

## Lifecycle & Ownership
- **Creation:** A concrete implementation of OAuthBrowserFlow is instantiated by a central authentication manager, such as an OAuthService, when a user initiates a login process that requires an external browser-based grant. It is created on-demand for a specific authentication attempt.
- **Scope:** The object's lifetime is strictly scoped to a single, active authentication session. It is a transient, short-lived object.
- **Destruction:** The instance is eligible for garbage collection as soon as the authentication attempt concludes, whether through success, failure, or timeout. The parent authentication service is responsible for releasing all references to it.

## Internal State & Concurrency
- **State:** This abstract class defines no state itself. However, its parent OAuthFlow and any concrete subclasses are inherently stateful, tracking tokens, nonces, and other details specific to the active authentication attempt. These objects are highly mutable.
- **Thread Safety:** This class is **not thread-safe**. Authentication flows are intrinsically tied to a specific user session and network connection. All interactions with an OAuthBrowserFlow instance must be confined to the single thread managing that user's session to prevent severe race conditions and state corruption.

## API Surface
The public contract is defined by the abstract methods that subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onFlowInfo(String info) | void | O(1) | Callback invoked by the authentication system to provide the necessary information (e.g., a URL) to be sent to the client for initiating the browser flow. |

## Integration Patterns

### Standard Usage
This class is not used directly. A developer implements a concrete subclass to handle a specific OAuth provider. The system's authentication service then drives the flow.

```java
// Example of a concrete implementation being driven by the system
// This code would exist within a central authentication manager.

// 1. A specific flow is instantiated for a user session.
OAuthBrowserFlow browserFlow = new MicrosoftOAuthBrowserFlow(userSession);

// 2. The system processes the flow, which eventually triggers the callback.
// Internally, this might make a request to Microsoft's servers.
authenticationService.startFlow(browserFlow);

// 3. The onFlowInfo method (implemented in the subclass) is called with the URL.
// The implementation would then send this URL to the game client.
// public void onFlowInfo(String url) {
//   userSession.sendPacket(new S2CBrowserFlowPacket(url));
// }
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not instantiate concrete subclasses manually. The central authentication service is responsible for managing the lifecycle of authentication flows.
- **State Re-use:** Never attempt to re-use an OAuthBrowserFlow instance for more than one authentication attempt. These objects are single-use and stateful.
- **Cross-Thread Access:** Do not share an instance of this flow across multiple threads. All operations must be synchronized with the user's session thread.

## Data Pipeline
This class is a key component in the pipeline that delivers an authentication URL from the server to the client.

> Flow:
> Client Login Request -> AuthenticationService -> **ConcreteOAuthBrowserFlow** instance created -> `onFlowInfo` callback triggered with URL -> Server Network Layer sends URL to Client -> Client opens URL in user's browser

