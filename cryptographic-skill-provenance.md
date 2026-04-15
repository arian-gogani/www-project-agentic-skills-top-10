---
layout: col-sidebar
title: "Control: Cryptographic Skill Provenance and Publisher Trust Anchoring"
---

## Summary

A cryptographic provenance layer for Agent Skills binds every published Skill to a verifiable publisher identity and makes the Skill's content tamper-evident. It is the supply-chain-security primitive on which host-side controls (permission policy, sandboxing, scanning) can compose. This control addresses five of the OWASP Agentic Skills Top 10 risks: **AST01, AST02, AST04, AST07, AST09.**

This document defines the control, states its scope and its limitations honestly, and shows how to implement it today using the Agent Skills specification's native `metadata` extension point.

---

## Risk context

As of April 2026 the Agent Skills ecosystem has:

- A format specification ([agentskills.io/specification](https://agentskills.io/specification))
- Tens of thousands of public Skills across `github.com/anthropics/skills` and 25+ third-party registries
- Zero built-in publisher-identity or content-integrity primitives
- At least one publicly reported compromise ("ClawHub registry" incident — multiple top-downloaded Skills confirmed as malware)
- Active security scanners (Cisco Skill Scanner, Snyk Agent Scan, SkillFortify, SkillScan) that detect content-level issues but cannot attest authorship

This situation is structurally analogous to where npm was before package signing, or where the web was before TLS Certificate Authorities. Without a publisher-identity primitive, hosts cannot distinguish a legitimate Skill from a substitution attack, and users cannot make informed trust decisions.

---

## Why signing alone is insufficient

A common misconception is that "signed = safe." It is not. A digital signature **proves who authored a given artefact**; it does not prove that the author is trustworthy or that the content is benign. An adversary can sign their own malicious Skill perfectly.

A complete trust chain therefore requires **four layers**:

| Layer | Question answered | Control type |
|---|---|---|
| **1. Identity** | Who authored this Skill? | Cryptographic signature |
| **2. Accountability** | Is the author a real, reachable entity? | Publisher verification (email, domain ownership, legal entity) |
| **3. Behavioural** | What does the Skill actually do? | Scanning (covered by Cisco / Snyk / SkillFortify / SkillScan) |
| **4. Reputational** | Does this author have a track record of non-malicious releases? | Trust registry with revocation |

A signature primitive **enables** layers 2 and 4 (they need something to bind to) and **composes with** layer 3 (scanners can annotate trust-registry records). It does not replace any of them.

This control covers layers 1 and 4, and specifies the integration contract for layers 2 and 3.

---

## The control

### C.1 — Signed manifest

Every Skill MUST have a cryptographic signature covering a canonical form of its `SKILL.md` and all files it declares as part of the Skill.

**Signing algorithm:** ECDSA P-256 with SHA-256 (JWA `ES256`) is recommended as the baseline. This matches the [OpenAPI `x-agent-trust` extension](https://spec.openapis.org/registry/extension/x-agent-trust.html) and [IETF `draft-sharif-agent-payment-trust`](https://datatracker.ietf.org/doc/draft-sharif-agent-payment-trust/), both of which are already registered for cross-ecosystem agent authentication.

**Canonicalisation:** the signed input is the `SKILL.md` content with signing-output fields (signature, digest, signedAt) stripped, line endings normalised to LF, trailing whitespace removed, plus a merkle-style hash over any declared resource files.

**Storage:** the signature and its binding metadata are stored as reverse-DNS keys inside the Skill's `metadata` field (the spec's sanctioned extension point), pending a first-class spec field.

### C.2 — Publisher identity binding

The signature MUST be bound to a resolvable publisher identity:

- A `keyid` naming the signing key, unique to the publisher
- A `publisher` field using a resolvable identifier (`did:web:example.com`, email, domain, or GitHub org)
- A `publicKeyUrl` where the verification key can be fetched (or a pinned key thumbprint for offline use)

Publisher verification (layer 2) is the registry's responsibility: email confirmation, domain ownership proof, corporate entity checks. This control specifies the binding, not the verification workflow.

### C.3 — Revocation

The control MUST support revocation of:

- A specific `keyid` (key compromise — all Skills signed under that keyid become untrusted)
- A specific Skill version by `canonicalDigest` (targeted content revocation)
- An entire publisher (accountability failure — all their Skills become untrusted)

Revocation is published by a trust registry via a CRL-style endpoint consulted by hosts at load time. Hosts SHOULD cache revocation state with a maximum freshness window (recommended: 24 hours for critical operations, 7 days for non-critical).

### C.4 — Host verification procedure

A conformant host, before loading a Skill into the agent's context:

1. Parse the `SKILL.md` frontmatter.
2. Extract the claimed digest, signature, publisher, and keyid.
3. Strip the signing-output fields from the frontmatter.
4. Compute the canonical digest; reject if it does not match the claimed digest.
5. Fetch or retrieve the pinned public key; reject if keyid does not resolve.
6. Verify the signature with the public key using the canonical input; reject on failure.
7. Query the trust registry: is this keyid or publisher revoked?; reject if so.
8. Only on success, inject the Skill's `description` into the agent's system prompt and permit the Skill's `allowed-tools`.

Rejection at any step MUST prevent the Skill's description from reaching the agent context, because the description itself is the prompt-injection attack surface (see AST04 mapping below).

---

## Implementation — native Agent Skills spec

The Agent Skills specification defines an arbitrary `metadata` extension point with the guidance *"Clients can use this to store additional properties not defined by the Agent Skills spec."* The control is implemented today via reverse-DNS-scoped metadata keys, pending a first-class `signature` spec field.

Minimum frontmatter fields for a conformant signed Skill:

```yaml
---
name: your-skill-name
description: (your description)
metadata:
  com.example.x-agent-trust.algorithm: "ES256"
  com.example.x-agent-trust.publisher: "did:web:your-domain.example"
  com.example.x-agent-trust.keyid: "your-skill-2026-04-15"
  com.example.x-agent-trust.publicKeyUrl: "https://your-domain.example/.well-known/your-skill.pub.pem"
  com.example.x-agent-trust.canonicalDigest: "sha256:<hex>"
  com.example.x-agent-trust.signature: "<base64 ECDSA signature>"
  com.example.x-agent-trust.signedAt: "2026-04-15T08:00:00Z"
---
```

Namespaces SHOULD use reverse-DNS of the publisher's controlled domain to prevent collisions.

## Reference implementations

Vendor-neutral examples of the control in production form:

- **Signing primitive (Apache 2.0):** [`x-agent-trust` on npm](https://www.npmjs.com/package/x-agent-trust), [GitHub reference](https://github.com/cybersecai-uk/x-agent-trust-reference)
- **Signed-Skill reference (Apache 2.0):** [`cybersecai-uk/agentpass-skill`](https://github.com/cybersecai-uk/agentpass-skill) — includes `sign.js`, `verify.js`, and a canonical `SKILL.md` with this control applied
- **Trust registry reference:** [AgentSearch](https://agentsearch.cybersecai.co.uk) — indexes Skills with L0–L4 publisher reputation and publishes a verification API
- **IETF Internet-Draft:** [`draft-sharif-agent-payment-trust`](https://datatracker.ietf.org/doc/draft-sharif-agent-payment-trust/) — specifies the underlying signature scheme

Any implementation that produces a conformant canonical form and ECDSA-P256 signature is interchangeable; the control defines the contract, not a specific library.

---

## AST mapping

| AST risk | How this control mitigates it |
|---|---|
| **AST01 Malicious Skills** | Publisher identity is cryptographically bound to every release. Malicious submissions can be traced to an accountable identity; anonymous distribution is disincentivised. Does NOT prevent a verified publisher from publishing malicious content — composes with scanning (AST08) and reputation (AST09). |
| **AST02 Supply Chain Compromise** | Signed canonical digest detects tampering post-publish, in transit, or during build. Any change to the `SKILL.md` or declared resources invalidates the signature. |
| **AST04 Insecure Metadata** | The `description` field — the primary prompt-injection surface — is covered by the canonical signed input. Any modification of the description after signing is detected at load time. |
| **AST07 Update Drift** | Signed binding of `keyid` + `signedAt` plus trust-registry revocation detects silent updates after initial trust, handles key rotation explicitly, and supports publisher takeover response. |
| **AST09 No Governance** | Publisher identity binding + revocation + reputation ladder (e.g. L0–L4) establish the accountability chain that governance requires. Trust registries become discoverable endpoints that hosts and downstream scanners integrate with. |

---

## Limitations and out-of-scope

This control **does not**:

- **Make a Skill safe to run** — a verified, signed publisher can still publish harmful content (see AST01 caveat above). Safety is a composite property requiring all four trust layers.
- **Replace host-side permission enforcement** (AST03 Over-Privileged Skills) — signatures attest authorship, not that the declared `allowed-tools` set is appropriate. Host-side sandboxing remains required.
- **Replace isolation** (AST06 Weak Isolation) — signed Skills still execute in the host environment; process and memory isolation is a host responsibility.
- **Solve cross-platform portability** (AST10 Cross-Platform Reuse) — a Skill signed for one host may behave differently on another; portability guarantees are out of scope.
- **Mandate a specific registry** — this control defines the signature contract; any conforming trust registry (commercial or community) can host publisher state.
- **Cover runtime-generated or agent-authored Skills** — this control addresses **statically-authored, pre-published** Skills. Skills an agent creates or modifies during execution, and sub-agents spawned by a parent agent (e.g. AutoGen orchestration, self-improving loops, code-generation-then-execute patterns), are out of scope for the publisher-signed model. They require a separate **delegation-chain control** specifying parent-signs-child semantics, capability-scope inheritance limits, delegation depth bounds, time-bound authority, signed delegation receipts, and revocation cascade. The underlying signing primitive (ECDSA P-256 / ES256) is the same; the trust anchor is the parent agent's signing key rather than a published publisher identity. A companion control for runtime-delegated Skills is a natural follow-up contribution and composes with this one.

Implementers should pair this control with runtime controls from the complementary OWASP guidance on agent sandboxing, permission enforcement, and behavioural monitoring.

---

## Integration with scanners (AST08)

Existing scanners (Cisco Skill Scanner, Snyk Agent Scan, SkillFortify, SkillScan) examine a Skill's content for malicious patterns, unsafe deserialization, and prompt-injection payloads. They are complementary to this control, not replaced by it:

- **Scanner output becomes a publisher-version-linked attestation** in the trust registry.
- **Unsigned Skills remain scannable** but carry an additional risk flag.
- **Signed Skills fail-closed on tampering** even if the scanner result is cached — signature verification precedes scan trust.
- **Revoked publishers** should force a scanner re-run across all their Skills before re-accreditation.

Registries SHOULD expose a single endpoint that combines signature-verification state, most-recent scanner output, and revocation status for a queried Skill.

---

## References

- [Agent Skills Specification](https://agentskills.io/specification)
- [OpenAPI Extensions Registry — `x-agent-trust`](https://spec.openapis.org/registry/extension/x-agent-trust.html)
- [IETF `draft-sharif-agent-payment-trust`](https://datatracker.ietf.org/doc/draft-sharif-agent-payment-trust/)
- [IETF `draft-sharif-mcps-secure-mcp`](https://datatracker.ietf.org/doc/draft-sharif-mcps-secure-mcp/)
- [OWASP MCP Security Cheat Sheet — Section 7 Message Integrity and Replay Protection](https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html)
- [OWASP MCP Top 10 — Recommended cryptographic security control](https://owasp.org/www-project-mcp-top-10/)
- [JWA algorithm `ES256`](https://datatracker.ietf.org/doc/html/rfc7518#section-3.1)

---

*Contributed by Raza Sharif, CyberSecAI Ltd — [contact@agentsign.dev](mailto:contact@agentsign.dev)*
