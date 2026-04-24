# Security Disclosure Policy

PebbleFlow takes security seriously, and we believe the people who find vulnerabilities in our products should be treated as collaborators, not adversaries. This document describes how to report a security issue, what's in scope, what to expect from us, and the safe-harbor commitments we make to good-faith researchers.

## Reporting a vulnerability

Please email **`security@pebbleflow.ai`**.

If your report contains sensitive technical detail, you may encrypt it using our PGP key (fingerprint and ASCII-armored key linked from [pebbleflow.ai/security.txt](https://pebbleflow.ai/.well-known/security.txt) once published).

Please include, where possible:

- A description of the issue and its impact.
- Steps to reproduce, or a proof-of-concept.
- Affected versions, platforms, or endpoints.
- Your name or handle, if you'd like credit.
- If illustrative, also submit a bug in app so that we can see logs and agent behavior.

You do not need a pre-existing relationship with us to file a report. You do not need to be a paying customer.

## What's in scope

- Anything in this repository (files mirrored from PebbleFlow's main codebase).
- The PebbleFlow extension on Chromium-based browsers and Safari.
- The PebbleFlow desktop applications (macOS, Windows, Linux).
- The PebbleFlow mobile applications (iOS, Android).
- The relay service at `relay.pebbleflow.ai`.
- The website at `pebbleflow.ai` (excluding third-party embedded services).

## What's out of scope

- Denial-of-service issues that rely on raw traffic volume rather than a logic bug.
- Findings on third-party services (Stripe, Cloudflare, Google APIs, Anthropic, OpenRouter, etc.) — please report those directly to the third party.
- Social-engineering attacks against PebbleFlow employees, contractors, or users.
- Physical attacks against PebbleFlow infrastructure, employees, or property.
- Vulnerabilities that require root or local administrative access to a user's device. (If a user is already root on their device, the threat model assumes they have access to their own data.)
- Issues in software you have modified or in environments you do not have authorization for.
- Reports generated solely by automated scanners without verification of impact.

## Our commitments to you

If you report a vulnerability in good faith and within the scope above:

1. **Acknowledgement within 2 business days.** A human will reply confirming we received your report.
2. **Initial assessment within 5 business days.** We will tell you whether we accept the report as a valid finding, request more information, or explain why we don't consider it in scope.
3. **Triage status within 14 days.** For accepted findings, we will share our planned remediation approach and a target timeline.
4. **Coordinated disclosure.** We ask that you give us a reasonable window (typically 90 days from acknowledgement, longer for complex issues) before public disclosure. We will not invoke this clause to indefinitely delay disclosure of a real issue.
5. **Credit at your discretion.** If you'd like to be named in our public disclosure or hall of fame, we will. If you'd like to remain anonymous, we will respect that.

## Safe harbor

We will not pursue legal action against, or support law-enforcement action against, a researcher who:

- Reports vulnerabilities in good faith and in accordance with this policy.
- Acts only against accounts and data that they own or have explicit permission to test.
- Does not exfiltrate, retain, or share user data beyond the minimum needed to demonstrate the issue.
- Does not deliberately degrade service availability for other users.
- Does not perform social-engineering attacks against PebbleFlow employees or users.
- Reports the issue to us before disclosing it publicly.

If a third party (a customer, a registrar, a hosting provider) takes action against you in connection with a good-faith report to us, please let us know and we will reach out to clarify on your behalf.

## Bug bounty

We do not currently run a paid bug bounty program. We may add one (likely on HackerOne or YesWeHack) once SOC 2 Type II is complete. If you would like to be notified when a bounty program launches, mention it in your report.

In the meantime, we offer credit, swag, and our genuine thanks. The security researchers who help us find issues are listed in [ACKNOWLEDGMENTS.md](ACKNOWLEDGMENTS.md) at their request.

## Vulnerability data we will not collect

We will not require you to identify yourself as a condition of accepting a report. We will not require you to sign a non-disclosure agreement before we will respond. We will not retain personal information about you beyond what's necessary to handle the report and credit you (if you've requested credit).

## Contact

- Security email: `security@pebbleflow.ai`
- General contact: `hello@pebbleflow.ai`
- Mailing address: Six Cailloux, LLC — see [pebbleflow.ai](https://pebbleflow.ai) for current address

We aim to respond to every report. If you have not heard from us within 5 business days, please re-send your report and CC `hello@pebbleflow.ai` in case the original was lost.
