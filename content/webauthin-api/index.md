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

## How to Integrate WebAuthn in Your Auth Stack (Frontend + Backend)

The WebAuthn API surface is small, but the data flowing between client and server requires careful handling. The browser returns `ArrayBuffer` objects that need encoding before transmission. The server receives attestation and assertion structures that must be parsed, validated, and stored correctly. Here is a practical walkthrough of both sides.

### Frontend: Registration

Registration starts with your server generating options (challenge, user info, relying party config) and ends with the browser returning a new credential.

```typescript
async function registerCredential(userId: string): Promise<void> {
  // 1. Fetch registration options from your server
  const options = await fetch('/api/webauthn/register/options', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ userId })
  }).then(res => res.json());

  // 2. Call the WebAuthn API
  const credential = await navigator.credentials.create({
    publicKey: {
      challenge: base64UrlToBuffer(options.challenge),
      rp: { id: options.rpId, name: options.rpName },
      user: {
        id: base64UrlToBuffer(options.userId),
        name: options.userName,
        displayName: options.userDisplayName
      },
      pubKeyCredParams: [
        { alg: -7, type: 'public-key' },   // ES256
        { alg: -257, type: 'public-key' }  // RS256
      ],
      authenticatorSelection: {
        userVerification: 'preferred',
        residentKey: 'preferred'
      },
      attestation: 'none',
      timeout: 60000
    }
  }) as PublicKeyCredential;

  // 3. Send the response to the server for verification
  const attestationResponse = credential.response as AuthenticatorAttestationResponse;
  await fetch('/api/webauthn/register/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      id: credential.id,
      rawId: bufferToBase64Url(credential.rawId),
      response: {
        clientDataJSON: bufferToBase64Url(attestationResponse.clientDataJSON),
        attestationObject: bufferToBase64Url(attestationResponse.attestationObject)
      },
      type: credential.type
    })
  });
}
```

### Frontend: Authentication

Authentication follows the same pattern. The server provides a challenge and optionally a list of allowed credential IDs, the browser collects the signed assertion, and the frontend forwards it.

```typescript
async function authenticate(): Promise<void> {
  const options = await fetch('/api/webauthn/login/options', {
    method: 'POST'
  }).then(res => res.json());

  const assertion = await navigator.credentials.get({
    publicKey: {
      challenge: base64UrlToBuffer(options.challenge),
      rpId: options.rpId,
      allowCredentials: options.allowCredentials?.map((cred: any) => ({
        id: base64UrlToBuffer(cred.id),
        type: 'public-key',
        transports: cred.transports
      })),
      userVerification: 'preferred',
      timeout: 60000
    }
  }) as PublicKeyCredential;

  const assertionResponse = assertion.response as AuthenticatorAssertionResponse;
  await fetch('/api/webauthn/login/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      id: assertion.id,
      rawId: bufferToBase64Url(assertion.rawId),
      response: {
        clientDataJSON: bufferToBase64Url(assertionResponse.clientDataJSON),
        authenticatorData: bufferToBase64Url(assertionResponse.authenticatorData),
        signature: bufferToBase64Url(assertionResponse.signature),
        userHandle: assertionResponse.userHandle
          ? bufferToBase64Url(assertionResponse.userHandle)
          : null
      },
      type: assertion.type
    })
  });
}
```

Both flows depend on a pair of encoding helpers. The WebAuthn API works with `ArrayBuffer`, but JSON transport requires base64url strings.

```typescript
function bufferToBase64Url(buffer: ArrayBuffer): string {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

function base64UrlToBuffer(base64url: string): ArrayBuffer {
  const base64 = base64url.replace(/-/g, '+').replace(/_/g, '/');
  const binary = atob(base64);
  return Uint8Array.from(binary, c => c.charCodeAt(0)).buffer;
}
```

### Backend: Parsing and Verification

The server's job is to unpack the response, verify it cryptographically, and reject anything that does not match. For registration, that means parsing `clientDataJSON` to confirm the challenge and origin, then extracting the public key from the `attestationObject`. For authentication, it means validating the signature over the `authenticatorData` and `clientDataJSON` hash against the stored public key.

Libraries like [`@simplewebauthn/server`](https://github.com/MasterKale/SimpleWebAuthn) handle the low-level CBOR decoding and signature math. Using one is strongly recommended over rolling your own verification.

```javascript
const {
  generateRegistrationOptions,
  verifyRegistrationResponse,
  generateAuthenticationOptions,
  verifyAuthenticationResponse
} = require('@simplewebauthn/server');

const RP_ID = 'example.com';
const ORIGIN = 'https://example.com';

// Registration verification
async function handleRegistrationVerify(req, res) {
  const { body } = req;
  const expectedChallenge = await getStoredChallenge(req.session.userId);

  const verification = await verifyRegistrationResponse({
    response: body,
    expectedChallenge,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID
  });

  if (verification.verified) {
    const { credential } = verification.registrationInfo;
    // Store the credential (see schema below)
    await saveCredential(req.session.userId, {
      credentialId: credential.id,
      publicKey: Buffer.from(credential.publicKey),
      counter: credential.counter,
      transports: body.response.transports || []
    });
  }
}

// Authentication verification
async function handleAuthVerify(req, res) {
  const { body } = req;
  const expectedChallenge = await getStoredChallenge(body.userId);
  const storedCredential = await getCredential(body.id);

  const verification = await verifyAuthenticationResponse({
    response: body,
    expectedChallenge,
    expectedOrigin: ORIGIN,
    expectedRPID: RP_ID,
    credential: storedCredential
  });

  if (verification.verified) {
    // Update counter to detect cloned authenticators
    await updateCredentialCounter(
      body.id,
      verification.authenticationInfo.newCounter
    );
    // Issue session
  }
}
```

Three fields inside `clientDataJSON` matter most during verification: `type` (must be `webauthn.create` for registration or `webauthn.get` for authentication), `challenge` (must match the server-issued value), and `origin` (must match your relying party origin exactly). If any of these fail, reject the response. Never trust `req.headers.origin` as your expected origin; hardcode it or load it from your server configuration.

### Credential Storage

Each user can register multiple credentials (a laptop fingerprint sensor, a phone, a backup security key), so your storage needs to support a one-to-many relationship between users and credentials.

```sql
CREATE TABLE webauthn_credentials (
  credential_id  TEXT PRIMARY KEY,
  user_id        TEXT NOT NULL REFERENCES users(id),
  public_key     BYTEA NOT NULL,
  counter        BIGINT NOT NULL DEFAULT 0,
  transports     TEXT[],           -- e.g. {'internal', 'usb'}
  created_at     TIMESTAMP DEFAULT NOW(),
  last_used_at   TIMESTAMP,
  friendly_name  TEXT              -- "MacBook Touch ID", "Backup YubiKey"
);
```

Track `last_used_at` so users can identify stale credentials in their account settings. Store `transports` so the browser can hint which authenticator type to prompt during login. Include a `friendly_name` column so users can distinguish between their registered devices. The `counter` column is your cloned-authenticator detection mechanism: if an authentication response arrives with a counter value less than or equal to the stored value, flag the credential and force re-registration.

## WebAuthn + Fallback and Progressive Enhancement Strategies

WebAuthn is the strongest authentication mechanism available in browsers today, but you cannot assume every user has access to it. Some users are on older devices without biometric sensors. Some are logging in from a borrowed machine. Some work in environments where USB security keys are impractical. A production auth flow needs to handle all of these cases without degrading security for the users who *can* use WebAuthn.

### Conditional Offering: Detect Before You Prompt

The right approach is progressive enhancement: offer WebAuthn when the device supports it, and fall back gracefully when it does not. The WebAuthn API provides the tools for this through feature detection.

```javascript
async function getAvailableAuthMethods() {
  const methods = ['password']; // baseline always available

  if (window.PublicKeyCredential) {
    methods.push('webauthn-roaming'); // security key support

    const platformAvailable =
      await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();

    if (platformAvailable) {
      methods.push('webauthn-platform'); // biometric/PIN support
    }
  }

  return methods;
}
```

`PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable()` tells you whether the device has a built-in authenticator like Touch ID, Face ID, or Windows Hello. If it returns `false`, prompting for a platform authenticator will only confuse the user. Check first, then adapt the UI accordingly.

Conditional mediation, introduced in WebAuthn Level 3, takes this further. With `navigator.credentials.get({ mediation: "conditional" })`, the browser can surface passkey options in the autofill dropdown alongside saved passwords. Users who have registered a passkey see it as a login option without any extra UI. Users who have not registered one see the standard password field. No special prompts, no feature flags on your end.

### Fallback Flows

When WebAuthn is unavailable or fails (the user cancels the prompt, a timeout occurs, the authenticator is not recognized), your application needs a clear fallback hierarchy. The goal is to maintain security while keeping the user moving forward.

A practical fallback chain looks like this:

1. WebAuthn (platform or roaming authenticator)
2. TOTP via authenticator app
3. One-time code via email or SMS
4. Pre-generated recovery codes
5. Administrative account recovery with identity verification

```javascript
async function handleAuthFallback(userId, failedMethod) {
  const registeredMethods = await getUserMFAMethods(userId);

  if (failedMethod === 'webauthn' && registeredMethods.totp) {
    return { fallback: 'totp', message: 'Enter the code from your authenticator app.' };
  }

  if (registeredMethods.backupCodes > 0) {
    return { fallback: 'backup', message: 'Enter one of your recovery codes.' };
  }

  return { fallback: 'recovery', message: 'Contact support to verify your identity.' };
}
```

Each step down the chain accepts a weaker proof of identity, so the tradeoffs matter. TOTP codes are still strong second factors. Email and SMS one-time codes are vulnerable to interception but are better than locking a user out entirely. Recovery codes are single-use secrets that should be generated at enrollment time, stored offline by the user, and hashed on the server. Administrative recovery should require out-of-band identity verification and should be logged and auditable.

The critical mistake is treating fallbacks as optional. Users will not register backup methods unless you build it into the enrollment flow. After a user registers their primary WebAuthn credential, prompt them immediately to set up a secondary method. Make it a required step, not a dismissible banner they will ignore.

### Matching Security to Risk

Not every action in your application carries the same risk. A user browsing a dashboard does not need the same authentication assurance as one changing account credentials or approving a financial transaction. Progressive enhancement applies here too.

For routine access, a valid session token is sufficient. For sensitive operations, trigger step-up authentication using the strongest method the user has registered. If they have a WebAuthn credential, require it. If they only have TOTP, require that. This layered approach lets you enforce strong authentication where it matters without adding friction to every interaction.

The underlying principle is straightforward: make the secure path the default path, design the fallbacks honestly, and never let the absence of one method leave a user without any path forward.
