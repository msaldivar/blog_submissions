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

