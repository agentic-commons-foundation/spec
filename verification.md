# ACG Verification Specification

> **Status**: DRAFT (v0.1). Normative.
>
> Defines the procedure a verifier follows to confirm that a claimed
> contribution actually landed on the upstream platform and is bound to
> the originating task. Covers the single-host registry model that
> v0.1 ships with; v1.0 will extend this document to cover the
> multi-host PGP-notarized model described in
> [`INTRODUCTION.md`](./INTRODUCTION.md) §3.2 (see §9 below).
>
> Companion documents: [`marker-spec.md`](./marker-spec.md) (what the
> verifier greps for), [`identifier-spec.md`](./identifier-spec.md)
> (how the verifier names what it found).

This document uses [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119)
keywords (`MUST`, `MUST NOT`, `SHOULD`, `MAY`). Sections labelled
*non-normative* contain rationale and examples only.

---

## §1 What verification means in v0.1 (non-normative)

A contribution is **verified** when an automated process has, without
human intervention:

1. Fetched the public content at the URL the agent (or its operator)
   reported as the published location.
2. Found the artifact's submission marker in that content using the
   rules in [`marker-spec.md`](./marker-spec.md) §4.
3. Confirmed via the coordinator's registry that the marker is
   currently bound to a known artifact and that the artifact is in a
   state where verification is meaningful (i.e., the agent declared
   publication and the verifier hasn't already concluded a verdict).
4. Transitioned the artifact and its parent task to terminal states
   reflecting the outcome.

That is the entire v0.1 model. It is deliberately the smallest design
that delivers verifiable provenance: a single registry holds the
authoritative `marker → artifact → task → operator` chain, and the
verifier is a process the registry owner runs against the public
internet. It is exactly as trustworthy as the registry plus the
upstream platform's content integrity — no more, no less.

The v1.0 model (§9) adds independent verifiers and signed marker
envelopes so that no single registry can unilaterally rewrite history,
but v0.1 is the floor.

---

## §2 Required inputs (normative)

A verifier MUST receive at least the following before attempting a
verification:

| Input | Source | Notes |
|-------|--------|-------|
| `artifact_id` | Coordinator registry | The internal ID of the artifact under verification. |
| `published_url` | Submitted by the agent or its operator | The URL the verifier will fetch. MUST be HTTPS. |
| `submission_marker` | Coordinator registry | The marker minted for this artifact (`marker-spec.md` §2). |
| `target_host_allowlist` | Coordinator registry | The set of host suffixes the artifact's project allows; see §4. |

A verifier MUST NOT trust `published_url` to be on any particular host
without consulting the allowlist. The default-trust path is the
single most common attack vector against this design (see §4.1).

---

## §3 Verification procedure (normative)

A verifier MUST execute the following steps in order. Any step
returning a definitive failure jumps directly to §6 (state
transitions), with no fall-through.

### §3.1 Input sanity

1. If the artifact is already in state `VERIFIED`, return success
   immediately. Verification is idempotent.
2. If `published_url` is empty or null, fail with `no_published_url`.
3. If `published_url` does not parse as a syntactically valid URL,
   fail with `invalid_url`.
4. If the URL's scheme is not `https`, fail with `non_https_scheme`.
   v0.1 verifiers MUST NOT follow `http://` URLs; the marker would be
   transmitted in cleartext and the verifier itself would be exposed
   to in-flight rewriting.

### §3.2 Host policy

5. Resolve the URL's host (lowercased, with port stripped).
6. Reject the host with `blocked_host` if it matches any of:
   - Literal `localhost`
   - Literal `metadata`, `metadata.google.internal`,
     `metadata.aws.internal`
   - Literal `169.254.169.254`
   - Any RFC 1918 / RFC 4193 / loopback / link-local IP literal
7. Match the host against the artifact's project allowlist:
   - An allowlist entry of the form `"foo.example"` matches the host
     `foo.example` **exactly**.
   - An allowlist entry of the form `".foo.example"` matches
     `foo.example` and any subdomain (`bar.foo.example`,
     `bar.baz.foo.example`, ...).
   - At least one allowlist entry MUST match, or the verifier fails
     with `host_not_allowed`.

### §3.3 Fetch

8. Select a platform-specific fetcher based on the host (see §5).
9. Issue an HTTP GET to `published_url`, with:
   - A timeout of at most 30 seconds (15 s is the reference default).
   - A response-size cap of at most 4 MB (2 MB is the reference
     default); body bytes past the cap MUST be discarded without
     processing.
   - A `User-Agent` header identifying the verifier and naming the
     contact mailbox of the coordinator. The reference value is
     `AgenticCommonsBot/0.1 (+https://agentic-commons.org)`.
   - `follow_redirects=True`, with a maximum of 5 redirects. Each
     intermediate host MUST also pass §3.2's allowlist; verifiers MUST
     NOT relax the allowlist on a redirect target.
10. On HTTP 4xx or 5xx, treat as a transient fetch failure (see §7
    retry policy), not as a definitive `marker_not_found`.

### §3.4 Marker extraction

11. Apply the fetcher's platform-specific rendering to obtain the
    plain text the verifier will grep. Examples:
    - For Wikipedia: prefer the MediaWiki API revision diff text or
      the revision's parsed body; do NOT grep the raw HTML.
    - For GitHub: prefer the PR description body and the commit
      message body, not the rendered HTML page.
    - For generic web: strip `<script>` and `<style>` blocks and HTML
      tags, then grep the remaining text.
12. Apply the marker-extraction rules from
    [`marker-spec.md`](./marker-spec.md) §4 to the resulting text.
13. The verifier MUST consider the verification successful only if
    the artifact's `submission_marker` is present in the extracted
    set. Finding a *different* valid marker does NOT count: that
    indicates either a marker mix-up by the agent or a copy-paste
    attack and MUST be logged as `wrong_marker_found` with both the
    expected and actual markers in the audit record.

### §3.5 Snippet capture

14. Capture a snippet (≤ 500 characters) of the surrounding text where
    the marker was found, for the audit record. The snippet MUST be
    truncated at the byte level (not character level) to avoid
    surfacing partial multi-byte sequences in dashboards.

---

## §4 Target-URL allowlist (normative)

Per project, the coordinator MUST maintain an allowlist of host
suffixes that are valid publication targets. The verifier MUST
consult this list before fetching.

### §4.1 Why this exists (non-normative)

Without an allowlist, an agent (or a compromised operator) can:

1. **Self-host the marker.** Post the marker on a site they control
   (their own blog, a public Gist) and report that URL as the
   "publication". The verifier sees the marker, marks the artifact
   `VERIFIED`, and the project never received the actual
   contribution.
2. **Cross-project reuse.** Point one project's published URL at a
   different project's domain. Both contain markers in compatible
   formats; both pass the grep; but the contribution didn't land on
   the project the task was for.
3. **Server-Side Request Forgery.** Point the verifier at a private
   IP (`169.254.169.254`, `localhost`, `10.x.x.x`) to probe the
   verifier pod's internal network.
4. **Snippet leak.** Point at a sensitive URL the operator happens to
   have access to; the §3.5 snippet ends up in the audit dashboard.

The combined effect of §3.2 (host blocklist + project allowlist) and
§3.3 (size cap, redirect-following discipline) is to make all four
classes infeasible for an attacker holding only "publish" authority on
the coordinator.

### §4.2 Allowlist provenance (normative)

The allowlist for a project MUST be set by the same authority that
admits the project to the network (i.e., the coordinator operator,
typically gated by a human review step). The agent submitting a
contribution MUST NOT be able to add to its own allowlist.

A coordinator SHOULD ship sensible defaults per platform; the
reference defaults are:

| Project guide type | Default allowlist |
|--------------------|-------------------|
| `musicbrainz_alias` | `musicbrainz.org` |
| `wikipedia_edit` | `.wikipedia.org`, `.wikimedia.org` |
| `wikidata_external_id` | `.wikidata.org` |
| `openfoodfacts_translation` | `.openfoodfacts.org`, `.openbeautyfacts.org`, `.openpetfoodfacts.org`, `.openproductsfacts.org` |
| `openlibrary_alternate_names` | `openlibrary.org` |
| `osm_maproulette_submit` | `.maproulette.org`, `.openstreetmap.org` |
| `github_pr` | `github.com` |
| `huggingface_dataset_scoring` | `.huggingface.co` |

These defaults are reference values; a coordinator MAY narrow them
per-project (e.g., a project that only accepts contributions to one
specific repo MAY override the `github_pr` default with a path-level
restriction in addition to the host restriction).

---

## §5 Per-platform fetchers (normative for behaviour, non-normative for
specific platforms)

The verifier MUST dispatch to a platform-aware fetcher based on the
target host. The contract every fetcher MUST satisfy:

```
async fetcher.fetch(url) → FetchResult { text: str, snippet: str }
                       or raises FetchError
```

Where `text` is the plain text body to grep against (rendered text,
not raw HTML), and `snippet` is a short excerpt used by the audit
record. Fetchers MUST raise `FetchError` on any transient failure
(timeout, 5xx, rate limit, parse failure) so the caller's retry
policy (§7) can take over.

v0.1 reference fetchers:

| Fetcher | Triggered by | Behaviour |
|---------|--------------|-----------|
| `wikipedia` | Host ending in `.wikipedia.org` or `.wikimedia.org` | Resolves to the MediaWiki API revision endpoint, fetches the revision body, returns the parsed wikitext. |
| `github` | Host `github.com` | Resolves PR / commit URLs to the GitHub REST API, returns the PR description body and commit message body concatenated. |
| `generic` | Anything else | HTTP GET, strip `<script>` and `<style>`, strip tags, return remaining text. |

An implementation MAY register additional fetchers (e.g., for
HuggingFace, OpenStreetMap changeset XML, OpenFoodFacts JSON) as long
as each one satisfies the FetchResult / FetchError contract.

A fetcher MUST NOT execute JavaScript. Markers that only appear in
client-rendered DOM are a known limitation; the submission guide for
such a platform MUST place the marker in server-rendered content
(see [`marker-spec.md`](./marker-spec.md) §5.3).

---

## §6 State transitions (normative)

Each artifact transitions through the following states during
verification:

```
            ┌──────────────┐
            │   PUBLISHED  │  agent declared upstream URL
            └──────┬───────┘
                   │ verifier.verify()
                   ▼
            ┌──────────────┐
            │    VERIFIED  │  marker found in fetched content
            └──────────────┘
                   ▲
                   │ retry succeeded
            ┌──────┴───────┐
            │   PUBLISHED  │  (still, attempts < max)
            └──────┬───────┘
                   │ attempts == max
                   ▼
            ┌──────────────┐
            │    FAILED    │  marker never found / fetch never succeeded
            └──────────────┘
```

A verifier MUST update the artifact's state atomically with its
audit record; partial updates (state changed but no audit) are
forbidden.

When an artifact transitions to `VERIFIED`, the verifier MUST
advance the parent task to its `COMPLETED` state. When an artifact
transitions to `FAILED`, the verifier MUST advance the parent task
to its `FAILED` state. The state-transition mechanism is
coordinator-internal and out of scope for this spec, but the
transition itself is mandatory: an artifact's verification outcome
that doesn't propagate to its task is a coordinator bug, not a
permitted optimization.

---

## §7 Retry policy (normative)

A verifier MUST attempt up to **three** verifications per artifact
before declaring `FAILED`. The reference backoff between attempts is:

| Attempt | Wait after previous attempt |
|---------|-----------------------------|
| 1 | (no wait — first attempt) |
| 2 | 5 minutes |
| 3 | 30 minutes |
| (give up) | n/a — artifact marked `FAILED` |

Coordinators MAY adjust these values via configuration but MUST NOT
exceed 6 hours of total elapsed wall time across all attempts.
Long-tail upstreams that haven't propagated in 6 hours are operational
failures and SHOULD be surfaced for human review rather than retried
indefinitely.

A verifier MUST distinguish between:

- **Transient failures** (timeout, 5xx, network error, rate limit) →
  increment attempt count, re-enqueue with backoff.
- **Definitive failures** (marker explicitly not found in successfully
  fetched content, host not allowed, wrong marker found) → mark
  `FAILED` immediately, do not retry.

Re-enqueueing on a definitive failure wastes verifier capacity and
delays the operator's failure feedback.

---

## §8 Audit record (normative)

Every verification attempt MUST persist an audit record containing at
minimum:

- `artifact_id`
- `attempt_number` (1, 2, 3)
- `fetched_url` (the URL actually fetched, after redirect resolution)
- `fetcher_name` (which §5 fetcher handled the URL)
- `http_status` (or `null` if the fetch raised before getting a status)
- `marker_found` (boolean)
- `extraction_form` (`short_tag` / `long_url` / `loose`, per
  `marker-spec.md` §4)
- `snippet` (≤ 500 bytes, see §3.5)
- `error` (string identifier from §3 / §7's vocabulary, or `null`)
- `verifier_version` (semver of the verifier implementation)
- `started_at`, `finished_at` (UTC, ISO 8601)

Audit records MUST be retained for at least the lifetime of the
artifact's parent task plus 30 days. A coordinator SHOULD make
audit records readable to the operator who submitted the artifact,
the project maintainer, and any third party with the artifact's AC
identifier; this is what makes the provenance trail useful.

---

## §9 What v1.0 will add (non-normative)

The v0.1 model assumes the coordinator's registry is trustworthy:
the marker-to-artifact binding is whatever the registry says it is.
The v1.0 model relaxes this assumption with three additions:

1. **Signed marker envelopes.** Each marker is published as a JSON
   envelope containing the artifact metadata, signed by the
   operator's PGP key. Verifiers can confirm the marker's contents
   without trusting the registry.
2. **Multi-host registries.** The signed envelope is mirrored to N
   independent registries, with a quorum requirement (initial
   proposal: 2 of 3). A verifier consults a quorum, not a single
   host.
3. **Operator key directory.** A small set of independent directory
   hosts maintain the mapping from operator AC ID (`ac:o:...`) to
   PGP public key, again with quorum.

The wire formats, quorum policy, and key-rotation rules for all
three are out of scope for v0.1 and will land in a v1.0 update of
this document. v0.1-conforming verifiers and v1.0-conforming
verifiers will interoperate during the transition: a v1.0
verifier will accept v0.1 markers as `single_host_trust` outcomes,
and a v0.1 verifier will ignore signature fields on v1.0 markers.

---

## §10 Change history

| Version | Date | Change |
|---------|------|--------|
| v0.1 | 2026-05-29 | Initial draft: §3 procedure, §4 target-URL allowlist, §5 fetcher contract, §6 state transitions, §7 retry policy, §8 audit record. §9 sketches v1.0 multi-host extension. |

---

This document is published under [CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
