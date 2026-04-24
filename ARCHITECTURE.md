# PebbleFlow Architecture: A Verification Walkthrough

This document explains, in plain prose, how PebbleFlow handles your Workspace OAuth tokens, your Workspace data, and the messages that pass through our infrastructure. Each claim is anchored to a specific file in this repository so you can verify it.

The high-level picture:

> **PebbleFlow is built around the principle that the workspace should run next to your data, not the other way around.** Your Workspace OAuth tokens are stored encrypted on your device. Your Workspace data — emails, calendar events, Drive files — flows directly from your device to Google. Our infrastructure is not in the data path. The relay we do operate is end-to-end encrypted on its message bus and runs as a separate, runtime-isolated Cloudflare Durable Object per user.

The rest of this document walks through that picture, explains the parts of it that are subtle, and is honest about the caveats.

---

## Pillar 1: Workspace OAuth tokens are stored encrypted on your device, not on our servers

When you connect a Google or Microsoft account in PebbleFlow, you'll see Google's standard OAuth consent screen. From a user's perspective, this looks identical to authorizing any other AI tool. The architectural difference is what happens to the resulting tokens.

**The proof, in code:** [`src/auth/connections-manager.ts`](src/auth/connections-manager.ts) is the module responsible for OAuth connections to external services (Gmail, Drive, Calendar, Microsoft 365, etc.). The `OAuthConnection` record — including `accessToken` and `refreshToken` fields — is persisted through the local storage adapter. There is no code path in this file that writes those fields to a central database or transmits them to our infrastructure for storage. The storage adapter is platform-aware: it uses Keychain on Apple platforms, Android Keystore on Android, and the browser's secure storage in the extension.

**What happens when the agent reads your inbox:** the agentic orchestrator pulls the Bearer token from the on-device storage adapter and makes an API call from your device directly to `gmail.googleapis.com` (or the equivalent endpoint for the relevant service). PebbleFlow's infrastructure is not in that request path. You can verify this by running a network monitor — see the network-traffic table on [pebbleflow.ai/trust](https://pebbleflow.ai/trust).

---

## Pillar 2: The relay's message bus is end-to-end encrypted

When PebbleFlow components on different devices need to coordinate — your browser extension talking to your desktop app, your phone talking to a desktop, the side-panel UI invoking a tool that runs natively on macOS — they do so through a relay we host at `relay.pebbleflow.ai`. The relay is the rendezvous point that lets your devices find each other across networks.

**The proof, in code:** [`relay/user-relay.ts`](relay/user-relay.ts) is the Cloudflare Durable Object that handles WebSocket connections. The file's own header documentation states:

> *"User Relay Durable Object — Manages WebSocket connections for a single user. Routes messages between the user's server(s) and client devices. Supports multiple server connections with priority-based primary election. Handles E2E encrypted messages (opaque to relay) and key exchange."*

The handlers for the `ENCRYPTED` message type (search the file for `'ENCRYPTED'`) are explicitly commented:

> *"Encrypted message to specific client (we can't read it)"*

Key exchange between your devices happens through the relay envelope — your devices exchange public keys via the `key_exchange` message type — but the relay never holds, requests, or sees the symmetric session key your devices derive. From the relay's perspective, every subsequent message is opaque ciphertext. It can route messages to the right destination, but it cannot decrypt them.

**What this rules out:** an attacker who compromises the relay infrastructure (or, hypothetically, compels us to surveil) cannot read the contents of the messages flowing through it, because the relay is not architecturally able to.

---

## Pillar 3: The relay is single-tenant per user, by construction

PebbleFlow's relay is built on Cloudflare Durable Objects. Each user gets their own `UserRelay` instance — a separate, runtime-isolated piece of compute and storage. There is no shared multi-tenant database holding every user's live session state, because there is no multi-tenant database at all in that role.

**The proof, in code:** [`relay/user-relay.ts`](relay/user-relay.ts) is a `class UserRelay extends DurableObject`. Cloudflare's DO runtime guarantees one isolated instance per object ID. Different users have different object IDs (derived from the user identifier in the routing layer), and instances do not share memory, storage, or compute. This is enforced by Cloudflare's runtime, not by application-level access control.

**What this rules out:** lateral movement from one compromised user's session to any other user's data. There is no shared backing store to pivot through.

---

## Pillar 4: PebbleFlow-held service credentials are encrypted at rest

We do hold a small number of *PebbleFlow-owned* OpenRouter credentials in D1 — not user-owned secrets, but credentials OpenRouter minted on *our* OpenRouter account (via their OAuth provisioning flow or via direct provisioning). They exist so we can meter each user's free-tier AI usage against our own account without the user needing to bring their own key. The distinction matters: these are operational service credentials belonging to PebbleFlow, not user secrets we are "storing on your behalf."

Because they are still credentials, we encrypt them at rest.

**The proof, in code:** [`relay/api/crypto.ts`](relay/api/crypto.ts) implements `encryptAtRest()` and `decryptAtRest()` using AES-GCM-256 with an HKDF-derived key from the server-side secret `PF_TOKEN_SECRET`. The random 12-byte IV is encoded alongside the ciphertext as `base64url(iv).base64url(ciphertext)`; `isEncryptedAtRest()` detects that shape so the code can distinguish encrypted values from legacy plaintext during migrations.

**Where encryption happens on the write path:** [`relay/api/openrouter-oauth.ts`](relay/api/openrouter-oauth.ts) is the only public write path into `pf_service_keys`. Every write calls `encryptAtRest(key, env.PF_TOKEN_SECRET)` *before* the database insert. The companion read path at the same file decrypts only after confirming the stored value matches the encrypted shape.

**Where the claim is annotated in the schema:** [`relay/db/schema.sql`](relay/db/schema.sql) carries inline `-- encrypted at rest with PF_TOKEN_SECRET; see relay/api/crypto.ts` comments on every credential column in `pf_service_keys`, `user_provisioned_keys`, and `provisioned_keys`, so a reader following the schema alone can chase the citation back to the primitive.

**What this rules out:** a PebbleFlow employee with D1 console access, or an attacker who gains D1 read access, cannot recover the OpenRouter credentials without *also* compromising the Worker-side `PF_TOKEN_SECRET`. The secret lives only as a Cloudflare Worker secret and never enters D1, the mirror, or any code path that writes to disk.

**What this does NOT claim:** encryption-at-rest is not a substitute for careful access control, and it does not protect the credential once it's decrypted at request time. It is a defense-in-depth measure against the specific failure mode of "raw database access leaks the keys."

---

## What is and isn't in our central database

We do operate a central database (Cloudflare D1) for the things that genuinely need to be central. Honesty matters here.

**The proof, in code:** [`relay/db/schema.sql`](relay/db/schema.sql) is the complete current schema — every table and column that exists. You can read it end-to-end in a few minutes.

The database stores:

- **Account identity:** your email, your hashed password if you use email/password login, the provider ID returned by Google/Apple/Microsoft if you use a social-login button.
- **Billing state:** Stripe customer ID, subscription tier, license key, subscription status.
- **Per-user API keys for AI inference providers** (OpenRouter specifically). These are inference-provider credentials — distinct from your Workspace OAuth tokens. They exist for users on our managed-credits flow and for users who bring their own OpenRouter key.
- **Device activation list** for license enforcement.
- **Audit logs** kept for SOC 2 evidence requirements.
- **A routing table for opt-in inbound webhooks** (WhatsApp, Telegram, etc.) — used only if you've configured those messaging integrations.

The database **does not** store:

- Your Workspace OAuth tokens (Gmail, Calendar, Drive, Microsoft 365). Those are on your device only.
- Your conversation content. Conversations live in your local storage; the orchestrator runs in your side panel.
- Your Workspace data (emails, calendar events, Drive files). Those are read directly from Google/Microsoft to your device, never passing through our infrastructure.
- The plaintext contents of any message that flows through the WebSocket relay. The relay sees ciphertext only.

A breach of our central database would expose account/billing identity, audit logs, and the OpenRouter inference-key column. It would not yield Workspace tokens, conversation content, or Workspace data, because those are not in there to take.

---

## The honest caveats

This is a different threat model, not a magic shield. The qualifiers we'd rather you hear from us than discover.

### The OAuth code-exchange briefly touches our relay

Google (and Microsoft, GitHub, Slack) require the OAuth `client_secret` to be presented at the token-exchange step. That secret cannot be shipped in client code, so our stateless relay attaches the `client_secret` and forwards your authorization code to Google in exchange for the access and refresh tokens. The tokens come back through the relay and are immediately returned to your device for storage. They are not persisted in our infrastructure.

**The proof, in code:** [`relay/api/oauth-token-handler.ts`](relay/api/oauth-token-handler.ts) is the relay-side handler. It is a stateless proxy: it accepts the authorization code from the device, calls the provider's token endpoint with the `client_secret`, and returns the token response to the device. There are no `INSERT` or `UPDATE` statements writing tokens to D1. The `client_secret` itself lives as a Cloudflare Worker secret, not in the database and not in any file in this repository.

**Why this is the way it is:** every native Google-integrated app you've ever used has a server component for this same reason. It's a Google OAuth constraint, not a PebbleFlow design choice. We acknowledge it because we'd rather name a small touchpoint up front than have someone discover it via packet capture later and conclude we were hiding it.

### Custom OAuth endpoints let you bypass our OAuth client entirely

PebbleFlow supports configuring custom OAuth endpoints. A user willing to provision their own OAuth client in Google Cloud Console (or the Microsoft Entra equivalent) can avoid PebbleFlow's OAuth client and our relay's exchange step altogether. Once configured, the auth flow is between your device and Google with PebbleFlow nowhere in the loop.

We don't surface this as a consumer-facing setting because most users don't need it — the default architecture already keeps your Workspace data off our servers. But it exists, and the implementation will be familiar to anyone who has set up an OAuth client in Google Cloud Console before.

### Device compromise still matters

If your laptop is owned, your tokens are owned. Our architecture moves the attack surface from "vendor's vault" to "your endpoint." That is a smaller and more defensible surface — there is no centralized target whose compromise scales to thousands of users — but it is not zero. The standard endpoint-hygiene practices (full-disk encryption, OS up to date, no random downloaded executables, etc.) still apply.

### Login-tier "Sign in with Google" is identity-only

If you sign into PebbleFlow itself with Google or Apple, that flow is identity-only and creates a Basic connection scoped to your account. It is not the same path as the Workspace-access flow described above and does not grant the AI access to your inbox or calendar.

### Supply-chain attacks on PebbleFlow itself are still conceivable

A compromised update channel could in principle ship code that exfiltrates tokens from your device. We mitigate this with code signing and notarization on every platform we ship — Apple's notarization on macOS and iOS, Google Play Protect on Android, Microsoft signing on Windows, the Chrome Web Store and Edge Add-ons review processes for the extension. The mitigation is different from the vendor-vault model, and so are the attackers' economics. A supply-chain attack that compromised our build pipeline would still need to ship a malicious binary that survived each platform's review, and the resulting exfiltration would happen one user-device at a time, not thousands at once from a central vault.

We do not claim this risk is zero. We claim it is a smaller and more attacker-economically-unfavorable risk than the SaaS-AI-vendor-holds-your-tokens-centrally model.

---

## Threat model summary

The architecture is designed to make these specific attack scenarios produce nothing useful:

| Scenario | Outcome under PebbleFlow's architecture |
|---|---|
| An attacker breaches our central database | Yields account identity, billing data, audit logs, and OpenRouter inference keys. Does **not** yield Workspace OAuth tokens, conversations, or Workspace data, because those aren't stored centrally. |
| An attacker compromises our relay | Can disrupt service availability. Cannot read the WebSocket message contents (E2E ciphertext only). Cannot pivot to read another user's data (per-user DO isolation). |
| An attacker compromises one user's Durable Object | Cannot use that compromise to access any other user's data. No shared backing store between users. |
| Compelled disclosure (subpoena, NSL, etc.) | We can produce account identity, billing data, audit logs, and OpenRouter keys. We cannot produce Workspace tokens, conversation content, or Workspace data, because we don't have them. We cannot decrypt message bus traffic, because we don't have the keys. |

The architecture does not protect against:

- A compromise of the user's device.
- A compromise of our update/release pipeline that ships a malicious client.
- Mistakes by the user (e.g., installing untrusted browser extensions that act on the same data).
- Bugs we haven't found yet. (See [SECURITY.md](SECURITY.md) for how to report them.)

---

## Cross-references to live verification

The companion blog post — [Not in the middle: why a Context AI–style breach against PebbleFlow would yield nothing useful](https://pebbleflow.ai/blog/not-in-the-middle-vercel-context-ai-breach) — includes a network-traffic table that shows what you'd see in Little Snitch, GlassWire, or Wireshark while using PebbleFlow normally. That table cross-references the architecture described here.

The trust page at [pebbleflow.ai/trust](https://pebbleflow.ai/trust) has an interactive data-flow diagram that visualizes the same architecture.

---

## Found something this document doesn't address?

Open an issue, or for security-relevant questions, follow the disclosure process in [SECURITY.md](SECURITY.md). We treat both seriously.
