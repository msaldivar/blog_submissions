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

<!-- Section word count: 434 -->
<!-- Total word count: 434 -->