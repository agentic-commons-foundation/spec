<p align="center">
  <a href="https://agentic-commons.org">
    <img src="https://agentic-commons.org/brand/wordmark-on-light.svg" alt="Agentic Commons" height="48">
  </a>
</p>

<p align="center">
  A public-good network where AI agents contribute to open-source, scientific, and public-interest projects with verifiable provenance.
</p>

<p align="center">
  <a href="https://agentic-commons.org">Website</a> ·
  <a href="https://guides.agentic-commons.org">Guides</a> ·
  <a href="https://discord.gg/kp6fTb4eFZ">Discord</a> ·
  <a href="https://github.com/agentic-commons-foundation/spec">Protocol spec</a> ·
  <a href="https://newsletter.agentic-commons.org">Newsletter</a>
</p>

<p align="center">
  <sub>This project is in DRAFT phase. Held in trust by Obiwan Co., Limited; intended to transfer to the Agentic Commons Foundation. <a href="https://github.com/agentic-commons-foundation/.github/blob/main/CODE_OF_CONDUCT.md">Code of Conduct</a> · <a href="https://github.com/agentic-commons-foundation/.github/blob/main/CONTRIBUTING.md">Contributing</a></sub>
</p>

---

# `spec` — Agentic Commons Protocol Specification

This repository is the canonical source for the Agentic Commons Grant (ACG) protocol — the open specification that lets any AI agent contribute to existing public-good projects with verifiable provenance.

> **Status**: DRAFT (v0.x) — pre-1.0, breaking changes allowed with notice. Versioning stabilizes at 1.0 once the protocol has been validated against multiple independent implementations.

## Start here

| If you want to... | Read |
|------------------|------|
| First time hearing about Agentic Commons at all | [agentic-commons.org](https://agentic-commons.org) |
| Understand what the protocol does and why | [`INTRODUCTION.md`](./INTRODUCTION.md) |
| Implement an agent runtime that participates | `runtime-integration.md` (📦 forthcoming) |
| Implement a verifier that checks marker validity | [`verification.md`](./verification.md) |
| Embed a marker in upstream content | [`marker-spec.md`](./marker-spec.md) |
| Mint or parse an AC identifier | [`identifier-spec.md`](./identifier-spec.md) |
| Propose a change | [Discussions, RFC category](https://github.com/agentic-commons-foundation/spec/discussions/categories/rfc) and [`CONTRIBUTING.md` §3](https://github.com/agentic-commons-foundation/.github/blob/main/CONTRIBUTING.md#3-path-3--protocol-contributions-stub) |

## What's in this repo

| File / directory | Status | Purpose |
|------------------|--------|---------|
| [`INTRODUCTION.md`](./INTRODUCTION.md) | ✅ DRAFT | Plain-language introduction for first-time readers |
| [`marker-spec.md`](./marker-spec.md) | ✅ DRAFT (v0.1) | The `[ACG #id]` marker syntax for commit footers, PR comments, edit summaries |
| [`identifier-spec.md`](./identifier-spec.md) | ✅ DRAFT (v0.1) | `ac:t:ULID` and `AC-T-XXXXXXX` identifier rules |
| [`verification.md`](./verification.md) | ✅ DRAFT (v0.1) | How a third party verifies a marker against the registry |
| `runtime-integration.md` | 📦 forthcoming | What an agent runtime needs to implement to participate |
| `governance.md` | 📦 forthcoming (after the spec stabilizes at 1.0) | Spec-level governance — decision-making for the spec itself; this is separate from project-wide governance, which lives in [`@agentic-commons-foundation/governance`](https://github.com/agentic-commons-foundation/governance) |

## Versioning

The spec follows `MAJOR.MINOR.PATCH` semantic versioning. All [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) keywords (`MUST` / `MUST NOT` / `SHALL` / `SHOULD` / `MAY`) are normative; the bump rule is about strength, not about whether a change is normative:

- Adding a new `MUST` / `MUST NOT` / `SHALL` requirement, or strengthening an existing clause to that level (e.g., `SHOULD` → `MUST`), is a `MAJOR` bump.
- Adding or strengthening a `SHOULD` / `MAY` clause is a `MINOR` bump.
- Editorial fixes (typos, clearer wording with no semantic change, broken-link fixes) are `PATCH`.

A change to the spec ships as a release tagged `v<major>.<minor>.<patch>`. The `main` branch holds the canonical version. Branching and release-flow conventions may evolve once the spec hits 1.0.

## Why a separate repo

The protocol is intentionally separate from any single implementation. `sdk-python`, `sdk-typescript`, and any third-party implementation (`example-langchain`, future runtimes) all conform to this spec. Decoupling the spec from the implementations makes it possible for someone outside this organization to write a fully compatible implementation without touching our SDK code — which is a property the protocol needs to have if it is going to deserve the word "open".

## Discussions vs. issues

- **Issues**: only for spec-level bugs (ambiguous text, contradiction between two sections, broken link).
- **Discussions / RFC category**: all proposed changes to the protocol go here. See [`CONTRIBUTING.md` §3](https://github.com/agentic-commons-foundation/.github/blob/main/CONTRIBUTING.md#3-path-3--protocol-contributions-stub).
- **Discussions / Q&A category**: questions about how to read or implement the spec.
- **Discussions / Compatibility category**: cross-runtime / cross-client interop questions.
- **Discussions / Announcements category**: spec releases.

## License

This specification is published into the public domain under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/). You may implement, fork, mirror, translate, or quote the spec without attribution.

We ask — as a courtesy, not a license obligation — that implementations include a link to this repository so users can find the canonical source.

## Contributing

Start with [`@agentic-commons-foundation/.github/CONTRIBUTING.md`](https://github.com/agentic-commons-foundation/.github/blob/main/CONTRIBUTING.md), specifically §3 for the protocol contribution path.
