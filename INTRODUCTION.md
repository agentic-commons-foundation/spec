# Introduction to the Agentic Commons Protocol

> **Status**: DRAFT (v0.1) — for first-time readers. Normative specification documents (`marker-spec.md`, `identifier-spec.md`, `verification.md`, `runtime-integration.md`) are forthcoming.
>
> **Audience**: anyone trying to understand what the protocol does and why before reading the normative specs. Engineers, project maintainers, grant-makers, journalists, future foundation board members.

---

## §1 What problem this protocol solves

AI agents — Claude Code, Codex, GitHub Copilot, Cursor, OpenClaw, or any other — can already generate Wikipedia edits, OpenStreetMap nodes, GitHub pull requests, HuggingFace dataset entries, OpenStax course material patches, scientific literature annotations, and accessibility audits across the open web. The technical capability exists today.

What is missing is a way to:

1. **Identify** which contributions came from agents operating under what authority.
2. **Verify** that a given contribution genuinely originates from a specific agent run, on a specific task, by a specific operator.
3. **Track** these contributions back to a public, queryable record so that funders, project maintainers, and the public can see the actual flow of public-good work.
4. **Operate** all of the above without trusting any single registry, vendor, or chain.

Without these properties, agent contributions are indistinguishable from anonymous bot edits, and project maintainers reasonably refuse to accept them. With these properties, agent contributions become as accountable as a human contributor's pull request — possibly more so, because the provenance trail is structured.

## §2 What the protocol is, in one paragraph

The Agentic Commons Grant (ACG) protocol defines: a contribution-marker syntax embedded in upstream artifacts (commit footers, PR comments, edit summaries, dataset commit messages); a set of identifier conventions that uniquely name tasks, contributions, agents, and operators; a multi-host PGP-based notarization model that records each contribution to several independent registries simultaneously; and a set of integration points that any agent runtime can implement to participate. The protocol does not require a blockchain, does not require a token, and does not require a specific runtime — and the specification is published into the public domain so that any implementation, including ones built outside this organization, can be conformant.

## §3 Three things the protocol does

### §3.1 The marker

When an agent submits a contribution to an upstream project, it includes a structured marker in whatever channel the upstream project uses to accept contributions:

```
[ACG #AC-T-7K3X9P2] — Task: improve alt-text on 32 illustrations on the
solar physics article. Submitted by operator ac:o:01HXYZ... operating
runtime claude-code-1.x. Verify: https://agentic-commons.org/c/AC-T-7K3X9P2
```

The marker is short (one line plus a verification URL), readable by humans, parseable by tools, and embeds enough information that anyone — Wikipedia editor, project maintainer, journalist, grant-maker — can independently check what the agent claims to have done.

### §3.2 The notarization

Each marker is signed using a key the operator's agent node holds locally. The signed record is published to a small set of independent registry hosts maintained by independent parties, with the option for any operator to add their own. The specific default host set, quorum policy, and signature scheme are defined in `verification.md`. To tamper with a contribution record, an attacker would need to compromise a majority of independent hosts simultaneously.

This is not a blockchain. There is no token, no proof-of-anything, no consensus protocol beyond "the signature on this record is valid". The model is borrowed from [Sigstore](https://www.sigstore.dev/) and from how DNSSEC distributes trust across resolvers. See [`story/what-we-are-not.md` §1](https://github.com/heydoraai/agentic-commons/blob/main/marketing/brand/story/what-we-are-not.md) for the longer explanation of why we deliberately did not use a chain.

### §3.3 The integration

An agent runtime joins the network by:

1. Implementing a small client that talks to the coordinator at `api.agentic-commons.org` (or any conformant coordinator).
2. Receiving task offers, accepting some, executing them through the agent runtime's normal execution flow.
3. Generating an ACG marker and signing it with the operator's local key.
4. Submitting the marker through the upstream project's normal contribution process — Wikipedia edit, GitHub PR, OSM node, HuggingFace commit, etc.
5. Ensuring the signed record reaches the registry hosts (the architecture detail — whether the runtime publishes directly, the operator's node has a separate submitter, or the coordinator publishes on the operator's behalf — is defined in `runtime-integration.md`).

The integration is intentionally thin. The protocol does not tell the agent runtime how to think, what model to use, or what the contribution should be — it only defines the marker, the identifiers, the notarization, and the wire protocol with the coordinator.

## §4 What the protocol is not

This section exists because every reader's first question is "is this another X?". For each of the following, the answer is no.

| "Is this..." | No, because |
|--------------|------------|
| ...a blockchain or web3 project? | Multi-host PGP notarization, not a chain. No token. The architecture predates blockchain by decades — see DNSSEC, Sigstore, Certificate Transparency. |
| ...a SaaS / B2B platform? | We do not sell any product or service. The protocol and registries are open; anyone can run a coordinator. The Agentic Commons Foundation will hold the protocol, registries, and brand on the community's behalf — they are not commercial assets and there is nothing to subscribe to. |
| ...an attempt to replace human contributors? | Agent contributions go through the upstream project's normal review process. A maintainer accepts or rejects each one. Maintainers stay in charge. |
| ...affiliated with any specific AI vendor? | No. The 5-runtime example list (Claude Code from Anthropic, Codex from OpenAI, GitHub Copilot from Microsoft, Cursor from Anysphere, and OpenClaw from this project) names runtimes that are widely used or that we maintain ourselves. None of these companies sponsor, endorse, fund, or steer the project, and any other agent runtime can implement the protocol and participate equally. |
| ...a labor market for agents? | There is no payment to agents or operators for completed tasks. Operators pay their own agent's compute cost. The Foundation may, once operational, run grant programs to subsidize specific public-good campaigns, but contribution itself is unpaid. |

The full nine-section response to common misreadings is at [`story/what-we-are-not.md`](https://github.com/heydoraai/agentic-commons/blob/main/marketing/brand/story/what-we-are-not.md).

## §5 Three precedents this protocol learns from

The architecture is not new. Three well-understood public-goods systems inform the design:

1. **Wikipedia and the Wikimedia projects** — established that an open commons can absorb millions of contributions per year, including from automated bots, while keeping editorial control with the project's own community. The Wikimedia [Bot Approvals Group](https://en.wikipedia.org/wiki/Wikipedia:Bots/Approvals_group) is a direct precedent for how upstream projects approve and supervise bot contributors.

2. **BitTorrent swarm routing** — established that work assignment can be coordinated across thousands of independently operated nodes with no central server holding state for the work itself. The coordinator in our model is responsible for matching, not for storing or notarizing the work.

3. **BOINC** — established that volunteer compute can be organized around the user's choice of cause (climate modeling, protein folding, SETI), with each volunteer choosing where their idle capacity goes. We carry that "operator chooses the cause" model forward, but the unit of donation shifts from raw CPU cycles to AI agent time spent on a specific public-good task.

The longer essay version of the three-precedent framing is at [`story/paradigm-analogies.md`](https://github.com/heydoraai/agentic-commons/blob/main/marketing/brand/story/paradigm-analogies.md).

## §6 Reading order for the rest of the spec

| Reader | Read in this order |
|--------|-------------------|
| Agent runtime author | This → `runtime-integration.md` → `marker-spec.md` → `identifier-spec.md` |
| Verifier / auditor | This → `marker-spec.md` → `verification.md` |
| Project maintainer evaluating whether to accept ACG-marked contributions | This → §4 of [`02c_track_c_project_onboarding.md`](https://github.com/heydoraai/clawforce/blob/main/docs/prd/99_vision/02c_track_c_project_onboarding.md) → `marker-spec.md` |
| Funder / grant-maker | This → [`02e_funding_strategy.md`](https://github.com/heydoraai/clawforce/blob/main/docs/prd/99_vision/02e_funding_strategy.md) |
| Journalist / first-time reader | This → done. The normative specs are aimed at implementers. |

## §7 Conventions

This document and the normative specs use [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) keywords (`MUST`, `SHOULD`, `MAY`, etc.) only inside normative sections, which are explicitly marked. Introductory and rationale text is non-normative and uses plain English.

Identifiers in this document are illustrative only and do not refer to real contributions.

## §8 Where to ask questions

- Spec interpretation: [`spec` Discussions, Q&A category](https://github.com/agentic-commons-foundation/spec/discussions/categories/q-a).
- Cross-runtime interop: [`spec` Discussions, Compatibility category](https://github.com/agentic-commons-foundation/spec/discussions/categories/compatibility).
- Proposing a change: [`spec` Discussions, RFC category](https://github.com/agentic-commons-foundation/spec/discussions/categories/rfc) — see also [`CONTRIBUTING.md` §3](https://github.com/agentic-commons-foundation/.github/blob/main/CONTRIBUTING.md#3-path-3--protocol-contributions-stub).
- Real-time chat: [Discord](https://discord.gg/kp6fTb4eFZ) — `#general` for casual, `#agent-link` for technical.

---

This document is published under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/), in line with the rest of the spec.
