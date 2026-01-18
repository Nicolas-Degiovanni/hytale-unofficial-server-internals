---
description: Architectural reference for OAuthClient
---

# OAuthClient

**Package:** com.hypixel.hytale.server.core.auth.oauth
**Type:** Transient Service

## Definition
```java
// Signature
public class OAuthClient {
```

## Architecture & Concepts
The **OAuthClient** is a high-level orchestrator for the Hytale server's interactions with the Hytale OAuth 2.0 identity provider. It is a critical component for establishing the server's identity and obtaining the necessary authorization tokens to communicate with protected backend APIs.

This class abstracts the significant complexity of two distinct, industry-standard OAuth 2.0 grant types:

1.  **Authorization Code Flow with PKCE (Proof Key for Code Exchange):** Implemented via the **startFlow(OAuthBrowserFlow)** method, this is the primary flow for interactive server setup. It dynamically creates a temporary local web server to act as a secure callback endpoint. This flow guides the server operator through a browser-based authentication and consent screen, using PKCE (RFC 7636) to mitigate authorization code interception attacks.

2.  **Device Authorization Flow:** Implemented via the **startFlow(OAuthDeviceFlow)** method, this flow is designed for headless environments or devices without a browser (e.g., a server accessed only via SSH). It provides the operator with a URL and a user code to be entered on a secondary device (like a smartphone or desktop), while the **OAuthClient** polls the token endpoint in the background until authorization is complete.

The client is responsible for all cryptographic operations (code verifier and challenge generation), state management (CSRF protection), HTTP request construction, and response parsing. It operates asynchronously, using callbacks (**OAuthBrowserFlow**, **OAuthDeviceFlow**) to deliver results without blocking the calling thread.

## Lifecycle & Ownership
-   **Creation:** The **OAuthClient** is a plain Java object created via its default constructor. It is expected to be instantiated by a higher-level service, such as an **AuthenticationManager** or during the server's initial bootstrap sequence.

-   **Scope:** While technically transient, an instance of **OAuthClient** should be treated as a long-lived, reusable service. It encapsulates a shared **HttpClient** instance, making repeated instantiation inefficient. A single instance should persist for the lifetime of the authentication management system.

-   **Destruction:** The object is managed by the Java garbage collector. There are no explicit cleanup or `close` methods. The temporary **HttpServer** used in the browser flow is automatically stopped upon completion, failure, or timeout of that specific flow.

## Internal State & Concurrency
-   **State:** The **OAuthClient** instance itself is stateless between method invocations. All state required for an authentication flow (e.g., CSRF tokens, code verifiers, temporary servers) is generated and managed entirely within the lexical scope of the `startFlow` methods and their asynchronous tasks. This design makes the client inherently clean and predictable.

-   **Thread Safety:** This class is thread-safe. Each call to a `startFlow` method initiates a new, independent, and isolated execution context. The shared internal **HttpClient** and **SecureRandom** instances are both designed for concurrent use. The use of **AtomicBoolean** for cancellation signals further confirms its suitability for multi-threaded environments.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| startFlow(OAuthBrowserFlow flow) | Runnable | Asynchronous | Initiates the browser-based Authorization Code Flow with PKCE. Returns a **Runnable** that, when executed, cancels the flow. |
| startFlow(OAuthDeviceFlow flow) | Runnable | Asynchronous | Initiates the Device Authorization Flow. Returns a **Runnable** that, when executed, cancels the flow. |
| refreshTokens(String refreshToken) | TokenResponse | Network-bound | **Synchronously** exchanges a refresh token for a new set of tokens. This is a blocking network call. Returns null on failure. |

## Integration Patterns

### Standard Usage
The client should be used asynchronously. The caller provides a callback interface implementation to handle success or failure and retains the returned **Runnable** to manage cancellation.

```java
// Assumes an AuthenticationService holds an OAuthClient instance
OAuthClient client = new OAuthClient();

// Define the callback handler
OAuthDeviceFlow deviceFlowHandler = new OAuthDeviceFlow() {
    @Override
    public void onFlowInfo(String userCode, String verificationUri, String verificationUriComplete, int expiresIn) {
        System.out.println("Please visit: " + verificationUri);
        System.out.println("And enter code: " + userCode);
    }

    @Override
    public void onSuccess(TokenResponse tokens) {
        System.out.println("Authentication successful!");
        // Persist tokens securely
    }

    @Override
    public void onFailure(String reason) {
        System.err.println("Authentication failed: " + reason);
    }
};

// Start the flow and store the cancellation handle
Runnable cancelHandle = client.startFlow(deviceFlowHandler);

// To cancel later:
// cancelHandle.run();
```

### Anti-Patterns (Do NOT do this)
-   **Discarding the Cancellation Handle:** Never call `startFlow` and discard the returned **Runnable**. This leaves an asynchronous task running with no mechanism for external cancellation, potentially leading to resource leaks or unexpected behavior on shutdown.
-   **Blocking on `refreshTokens` on a UI Thread:** The **refreshTokens** method is a blocking network call. Calling it from a time-sensitive thread (like a UI thread or main game loop) will cause the application to freeze. Execute it on a dedicated worker thread.
-   **Frequent Instantiation:** Do not create a `new OAuthClient()` for every authentication request. This is inefficient as it will create a new **HttpClient** instance each time. Maintain a single, long-lived instance.

## Data Pipeline
The data flow for the browser-based authorization process is a multi-step exchange between the server, the user's browser, and the Hytale identity provider.

> Flow:
> Server Startup -> **OAuthClient.startFlow()** -> Local HTTP Server Starts -> Auth URL Generated -> User opens URL in Browser -> User Authenticates with Hytale -> Hytale IDP redirects to Local HTTP Server with Auth Code -> **OAuthClient** receives code, exchanges it for tokens -> **OAuthClient** invokes `onSuccess` callback -> Authentication Service stores tokens

