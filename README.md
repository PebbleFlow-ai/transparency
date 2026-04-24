# PebbleFlow Transparency

This repository publishes the security-critical files from PebbleFlow's main codebase so that anyone can read, audit, or independently verify the privacy and security claims on [pebbleflow.ai/trust](https://pebbleflow.ai/trust).

PebbleFlow is a powerful, privacy-first workspace with an agentic orchestrator and chat interface that runs in a side panel. Our trust posture is structural — it holds because of how the code is written, not because of what we promise. This repo lets you check that for yourself.

## What's in this repo

A curated subset of files mirrored automatically from our main (private) repository:

| Directory | What it proves |
|---|---|
| `src/auth/connections-manager.ts` | Workspace OAuth tokens are stored encrypted on the user's device, not on our servers. |
| `src/auth/google-oauth.ts` | The shape of the Google login + Workspace-access OAuth flow. |
| `relay/user-relay.ts` | The relay's WebSocket message bus is end-to-end encrypted. The relay routes ciphertext between your devices but cannot read it. |
| `relay/api/oauth-token-handler.ts` | The OAuth code-exchange handler is a stateless proxy. It attaches the OAuth client_secret (which Google requires to live server-side), exchanges the code, and returns tokens to the device. It does not persist tokens. |
| `relay/api/crypto.ts` | The encrypt-at-rest primitives. AES-GCM-256 with HKDF-derived key from the server-side `PF_TOKEN_SECRET`. This is how we protect the PebbleFlow-owned service credentials that live in D1. |
| `relay/api/openrouter-oauth.ts` | The OpenRouter OAuth-delegated provisioning flow. Shows the write path that calls `encryptAtRest(…)` before inserting into `pf_service_keys`, and the read path that decrypts on fetch. |
| `relay/db/schema.sql` | The current, consolidated database schema. Every table and column that exists, and by omission every one that doesn't. No columns exist for Workspace OAuth tokens, conversations, or Workspace data. Credential columns carry inline `-- encrypted at rest` comments citing `relay/api/crypto.ts`. |
| `ARCHITECTURE.md` | A narrative walk-through of the privacy and security architecture, with code references back to the files above. |

## What's NOT in this repo, and why

- **The agentic orchestrator, the side-panel UI, the modes, and the prompts.** These are the product. Open-sourcing them is a separate decision we may revisit later, but they're not what's load-bearing for the privacy claims.
- **Non-security-relevant relay code.** Stripe webhooks, license rotation, billing webhooks, etc. They're proprietary and they don't touch your Workspace data.
- **Internal admin tooling, CI configuration, deploy scripts, secrets management.** Out of scope on principle.
- **The benchmark pipeline, the website, the marketing assets.** Not security-relevant.

If you think a file should be in scope but isn't, open an issue and tell us why.

## How this repo is kept current

A GitHub Action in our private monorepo mirrors the in-scope files into this repo on every push to `main`. Drift between the main codebase and this repo should be at most one push cycle. If you ever see a mirrored file with a stale `Last-Mirrored-At` timestamp at the top, please open an issue.

If you have a fix for security-relevant code, please follow the disclosure process in [SECURITY.md](SECURITY.md).

## How to read this repo

The shortest path:

1. Read [ARCHITECTURE.md](ARCHITECTURE.md). It walks the architecture pillar-by-pillar with line-anchored references into the mirrored files.
2. Cross-check the claims against the actual files. Each section of ARCHITECTURE.md cites specific symbols and line ranges.
3. If you want to verify the live behavior, see the "Verify it yourself" section of [pebbleflow.ai/trust](https://pebbleflow.ai/trust) — it includes a network-traffic table you can reproduce with Little Snitch, GlassWire, or Wireshark.

## Reporting a security issue

See [SECURITY.md](SECURITY.md). The short version: email `security@pebbleflow.ai`. We commit to acknowledgement within 2 business days and an initial assessment within 5.

## License

The mirrored code in this repo is licensed under [Apache 2.0](LICENSE). The documentation (`ARCHITECTURE.md`, this `README.md`, etc.) is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

The Apache license here is for the mirrored excerpts. PebbleFlow's main codebase remains proprietary under separate terms.

## Contact

- Security issues: `security@pebbleflow.ai`
- General inquiries: `hello@pebbleflow.ai`
- Website: [pebbleflow.ai](https://pebbleflow.ai)
- Trust page: [pebbleflow.ai/trust](https://pebbleflow.ai/trust)
