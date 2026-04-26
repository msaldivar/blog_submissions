# What Is the WebAuthn API?

The Web Authentication API (WebAuthn) is a [W3C standard](https://www.w3.org/TR/webauthn-3/) that lets web applications authenticate users with public-key cryptography instead of passwords. It is the browser-facing half of the [FIDO2 framework](https://fidoalliance.org/fido2-2/fido2-web-authentication-webauthn/), which pairs WebAuthn with the Client-to-Authenticator Protocol (CTAP) to enable passwordless, phishing-resistant login across browsers and platforms. WebAuthn reached W3C Recommendation status in March 2019, and Level 3 of the spec entered Candidate Recommendation in January 2026 with support across Chrome, Firefox, Edge, and Safari.

The problem WebAuthn solves is structural. Password-based authentication relies on shared secrets: the server stores a representation of your password, and you prove your identity by reproducing it. That model breaks in predictable ways. Credential stuffing, password spraying, and database breaches all exploit the same fundamental weakness. Even layering traditional MFA on top only raises the bar; it doesn't change the underlying architecture. Authenticator app codes and SMS tokens are still shared secrets that can be intercepted through phishing, SIM swapping, or social engineering.

WebAuthn eliminates shared secrets entirely. When a user registers with a WebAuthn-enabled service, their device generates an asymmetric key pair. The private key stays on the device, locked inside a secure element or trusted platform module. The public key goes to the server. During authentication, the server issues a cryptographic challenge that only the correct private key can sign. No password crosses the wire. No secret is stored on the server to be stolen.

Four core concepts define how WebAuthn operates:

**Credential.** A credential is the public-key pair bound to a specific user and a specific origin (the website's domain). Each credential is scoped so it can only be used on the origin where it was created. This origin binding is what makes WebAuthn phishing-resistant; a cloned login page at a different domain simply cannot trigger the correct credential.

**Attestation.** During registration, the authenticator can provide an attestation statement: a cryptographic proof that the credential was created by a genuine, trusted device. Attestation lets the server verify the authenticator's make and model, which matters in regulated environments where only hardware-backed keys are acceptable.

**Assertion.** During login, the authenticator produces an assertion: a signed response to the server's challenge that proves the user possesses the private key. The server validates this signature against the stored public key to complete authentication.

**Origin binding.** The browser enforces that authentication requests are scoped to the exact origin (scheme + domain + port) that registered the credential. Even a pixel-perfect phishing site at a different URL will fail because the browser will not match credentials across origins. This protection operates at the protocol level, independent of user behavior or awareness.

Together, these mechanisms create an authentication model where there is nothing to phish, nothing to replay, and nothing stored on the server worth stealing.

## How WebAuthn Works: Registration and Authentication Flows

WebAuthn operates through two ceremonies: registration (creating a credential) and authentication (proving you own it). Both follow the same pattern. The server generates a challenge, the browser mediates the interaction with the authenticator, and the authenticator performs the cryptographic operation. Understanding these flows is essential before writing any integration code.

### Registration with `navigator.credentials.create()`

Registration binds a new credential to a user account. The server initiates the process by generating a random challenge and specifying parameters about the relying party (your application) and the user.

```javascript
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: serverGeneratedChallenge,       // random bytes from your server
    rp: { id: "example.com", name: "My App" }, // relying party identity
    user: {
      id: userId,                              // unique user handle (ArrayBuffer)
      name: "user@example.com",
      displayName: "Jane Developer"
    },
    pubKeyCredParams: [
      { alg: -7, type: "public-key" },         // ES256 (recommended)
      { alg: -257, type: "public-key" }        // RS256 (broad compatibility)
    ],
    authenticatorSelection: {
      userVerification: "preferred"             // biometric/PIN if available
    },
    attestation: "none"                         // or "direct" for device verification
    timeout: 60000
  }
});
```

When this call executes, the browser prompts the user to interact with an authenticator: a fingerprint sensor, a security key tap, or a platform prompt like Windows Hello. The authenticator generates a new key pair, locks the private key inside its secure storage, and returns the public key along with a credential ID. Your frontend sends this response to the server for verification and storage.

### Authentication with `navigator.credentials.get()`

Authentication proves the user controls the private key registered earlier. The server sends a challenge and optionally a list of acceptable credential IDs.

```javascript
const assertion = await navigator.credentials.get({
  publicKey: {
    challenge: serverGeneratedChallenge,
    rpId: "example.com",
    allowCredentials: [{
      id: storedCredentialId,                   // from registration
      type: "public-key",
      transports: ["usb", "ble", "internal"]   // hint for authenticator discovery
    }],
    userVerification: "preferred",
    timeout: 60000
  }
});
```

The authenticator locates the matching credential, verifies the user (biometric or PIN), and signs the challenge with the private key. The browser returns this signed assertion to your application, which forwards it to the server for signature verification.

### The Backend's Role

The server has three responsibilities across both ceremonies.

During registration, the backend must verify the attestation response. This means confirming that the challenge matches what was issued, the origin matches your application's domain, and the public key uses an acceptable algorithm. Libraries like `@simplewebauthn/server` handle the cryptographic verification, but your application is responsible for storing the resulting credential data: the public key, credential ID, a signature counter, and the transports the authenticator supports.

During authentication, the backend verifies the assertion signature against the stored public key. It also checks the signature counter. Authenticators increment this counter with each use, so a counter value lower than or equal to the stored value signals a potential cloned credential. After successful verification, the backend updates the stored counter and issues a session.

The step-by-step flow looks like this:

1. Server generates and stores a random challenge
2. Frontend calls `navigator.credentials.create()` or `.get()` with the challenge
3. Browser mediates the authenticator interaction, enforcing origin binding
4. Authenticator performs the cryptographic operation after user verification
5. Frontend sends the response to the server
6. Server validates the response (challenge, origin, signature, counter)
7. Server stores the credential (registration) or issues a session (authentication)

Origin binding happens at step 3. The browser automatically includes the current origin in the data sent to the authenticator. If a user is on a phishing site at `evil-example.com`, the authenticator will not find a credential registered for that origin. The authentication fails silently, with no secret exposed to the attacker.

## WebAuthn Spec and Browser Support

### The W3C Specification

The WebAuthn standard has matured through three specification levels. [Level 1](https://www.w3.org/TR/webauthn-1/) became a W3C Recommendation in March 2019, establishing the core API for credential creation and assertion. [Level 2](https://www.w3.org/TR/webauthn-2/) followed in April 2021, adding support for cross-platform authenticator transports, improved attestation handling, and better alignment with the CTAP2 protocol. [Level 3](https://www.w3.org/TR/webauthn-3/) reached Candidate Recommendation status in January 2026 and introduces features like the PRF extension for deriving encryption keys from credentials, conditional mediation (the ability to autofill passkey prompts), and the Signal API for relying parties to communicate credential state changes back to authenticators.

Each level builds on the previous one without breaking backward compatibility. If your application targets Level 2 features today, it will continue to work as Level 3 gains full browser adoption.

### Browser Compatibility and Secure Contexts

WebAuthn enjoys broad support across modern browsers. Chrome has supported it since version 67, Firefox since version 60, Edge since version 18, and Safari since version 13. On mobile, both Android (via Chrome and the platform credential manager) and iOS (via Safari with Face ID and Touch ID) provide full WebAuthn support. The practical result is that the vast majority of users on current browsers can use WebAuthn credentials without installing anything.

One hard requirement: WebAuthn only works in [secure contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts). That means HTTPS in production, no exceptions. The API will not execute over plain HTTP. For local development, `localhost` is treated as a secure context by browsers, so you can test without configuring TLS certificates on your dev machine. This requirement exists because origin binding is the foundation of WebAuthn's phishing resistance, and origin verification is only meaningful over authenticated connections.

Browser support is not perfectly uniform across every feature. Platform authenticator availability depends on the operating system (Windows Hello, macOS Touch ID, Android biometrics), and newer Level 3 extensions like PRF are still rolling out across browsers and platforms. Implement feature detection using `PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()` and provide graceful fallbacks for users on older configurations.

### From U2F to WebAuthn

WebAuthn did not appear in a vacuum. It evolved from FIDO U2F (Universal 2nd Factor), a protocol the FIDO Alliance introduced in 2014 for hardware-based second-factor authentication. U2F was effective but limited: it only functioned as a second factor alongside a password, it required a physical USB security key, and it never saw native Safari support.

WebAuthn addresses each of those limitations. It supports both multi-factor and single-factor (passwordless) authentication, so a fingerprint scan or security key tap can replace passwords entirely rather than just supplementing them. It works with platform authenticators built into devices, not only roaming USB keys. And it has universal browser adoption, something U2F never achieved.

The transition is effectively complete. Chrome deprecated the U2F JavaScript API in version 98 and removed it entirely in version 115. Firefox deprecated U2F in favor of WebAuthn starting with version 60. Existing U2F security keys remain compatible with WebAuthn through CTAP1 backward compatibility, so users with older YubiKeys or Google Titan keys do not need to replace their hardware. For new implementations, WebAuthn through the FIDO2 framework is the only standard worth building on.

