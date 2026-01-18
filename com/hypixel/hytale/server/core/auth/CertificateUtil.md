---
description: Architectural reference for CertificateUtil
---

# CertificateUtil

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Utility

## Definition
```java
// Signature
public class CertificateUtil {
```

## Architecture & Concepts
The CertificateUtil class is a critical, stateless security component within the server's authentication and authorization framework. Its primary function is to enforce **Certificate-Bound Access Tokens**, a high-security pattern that cryptographically links a client's authentication token (JWT) to their underlying mTLS session certificate.

This utility acts as the bridge between the transport layer security (mTLS) and the application layer authentication. By verifying that the fingerprint of the certificate presented during the TLS handshake matches a specific claim within the JWT, it mitigates sophisticated attacks, including token theft and replay. If an attacker compromises a JWT, it is useless without the corresponding client's private key, which is required to establish the mTLS session.

This class centralizes all cryptographic operations related to certificate fingerprinting and comparison, ensuring consistent and secure implementation across the server.

### Lifecycle & Ownership
As a static utility class, CertificateUtil does not have a traditional object lifecycle.

- **Creation:** The class is never instantiated. It is loaded into the JVM by the ClassLoader when its static methods are first invoked, typically by the server's authentication filter.
- **Scope:** Application-level. Its methods are available globally throughout the server's runtime.
- **Destruction:** The class is unloaded from memory when the JVM terminates. No manual cleanup is required.

## Internal State & Concurrency
- **State:** CertificateUtil is entirely **stateless**. It maintains no internal state across method invocations. Each method operates exclusively on the arguments provided.
- **Thread Safety:** This class is unconditionally **thread-safe**. Its methods are pure functions with no side effects or reliance on shared mutable state. It can be safely and concurrently invoked from any number of threads, such as worker threads handling simultaneous client connections, without requiring any external synchronization or locks.

## API Surface
The public API consists of static methods for performing security-critical cryptographic operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| computeCertificateFingerprint(X509Certificate) | String | O(1) | Computes the SHA-256 hash of a certificate and returns it as a Base64URL-encoded string. Returns null on failure. |
| validateCertificateBinding(String, X509Certificate) | boolean | O(1) | The core security function. Compares a JWT fingerprint claim against the actual client certificate's fingerprint. |
| timingSafeEquals(String, String) | boolean | O(N) | Compares two strings using a constant-time algorithm to prevent timing-based side-channel attacks. |

**Warning:** The boolean return value of `validateCertificateBinding` is a critical security decision point. Failure to act upon a `false` result will neutralize the entire certificate binding security measure.

## Integration Patterns

### Standard Usage
CertificateUtil should be invoked within a security filter or interceptor that runs early in the request processing pipeline. The filter must have access to both the client's mTLS certificate and their authentication token.

```java
// Example from within a network authentication handler
public boolean handleAuthentication(RequestContext context) {
    X509Certificate clientCert = context.getTlsSession().getClientCertificate();
    String jwtFingerprint = context.getJwt().getClaim("cnf.x5t#S256");

    // Delegate the cryptographic validation to the utility
    boolean isBindingValid = CertificateUtil.validateCertificateBinding(jwtFingerprint, clientCert);

    if (!isBindingValid) {
        // REJECT the request immediately
        return false;
    }

    // Proceed with further authentication checks
    return true;
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new CertificateUtil()`. This class is designed for static invocation only.
- **Insecure Comparison:** Do not bypass `timingSafeEquals` by using standard `String.equals()` for comparing fingerprints. Doing so introduces a timing attack vulnerability that an attacker could exploit to guess a valid fingerprint.
- **Ignoring Failures:** The `computeCertificateFingerprint` method can return null if the SHA-256 algorithm is unavailable or the certificate is malformed. Code must handle this null case and treat it as a validation failure.

## Data Pipeline
CertificateUtil sits at the decision point where transport-layer identity (the certificate) is verified against application-layer identity (the JWT).

> Flow:
> mTLS Handshake -> X509Certificate -> Authentication Filter -> JWT Fingerprint Claim -> **CertificateUtil.validateCertificateBinding** -> Authorization Decision -> Request Processor

---

