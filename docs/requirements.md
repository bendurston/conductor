# Conductor — Requirements

> **Audience:** This document is the agent-facing source of truth for what Conductor must do and
> the constraints it operates under. Human readers should start with the [README](../README.md).
>
> **Scope:** Requirements only. This document deliberately does **not** prescribe an
> implementation architecture (module layout, class interfaces, package structure). The project's
> shared vocabulary lives in [`CONTEXT.md`](../CONTEXT.md); hard-to-reverse decisions and the
> reasons behind them live in [`docs/adr/`](./adr/). This document references both rather than
> restating them.
>
> Requirement keywords **MUST**, **SHOULD**, and **MAY** are used in the sense of RFC 2119.
> Requirements are numbered (`FR-n`, `NFR-n`) so issues, tests, and later docs can reference them.

---

## 1. Overview

Conductor is a command-line tool that safely copies a user's music library from one streaming
service to another (for example, Spotify → YouTube Music). It is **privacy-first**,
**config-driven**, and **bring-your-own-credentials**: the user supplies their own API credentials
for each service, and their library data and credentials never leave their machine except in calls
to the services' own APIs. The tool is aimed at individuals who want to move between services
without trusting a third-party hosted service with their accounts.

---

## 2. Goals & Non-Goals

### Goals

- Safely transfer a music library between two supported services.
- Preserve user privacy — no central storage of credentials or library data.
- Be easy to use — a single config file and a clear, actionable report.
- Be config-driven — behavior is determined by a user-provided configuration.
- Be extensible — new services can be added over time.

### Non-Goals

- **Not** a hosted or cloud service; Conductor runs locally on the user's machine.
- **Not** a credential store; Conductor never persists or centralizes user credentials.
- **Not** a music downloader or piracy tool; it transfers *library metadata* (what is saved), not
  audio files.
- **Not** a continuous background sync; a transfer is a user-initiated, one-directional operation.

---

## 3. Terminology

The project's canonical vocabulary — Service, Source, Destination, Transfer, Dry run, Capability,
Library item type, Playlist, Saved songs, Saved album, Saved artist, Match, Confidence threshold,
Ledger, and Retry store — is defined in **[`CONTEXT.md`](../CONTEXT.md)**. This document uses those
terms as defined there.

---

## 4. Functional Requirements

- **FR-1 — Service-to-service transfer.** Conductor MUST transfer a music library between two
  supported services, in a single direction (source → destination) per transfer.
- **FR-2 — User-supplied credentials.** Conductor MUST authenticate to each service using the
  user's **own** credentials. It MUST NOT ship or rely on shared/embedded credentials. The concrete
  authentication method for each service is intentionally left open at this stage; each supported
  service requires **some** method(s) of authentication, and a transfer requires credentials
  configured for **both** its source and destination (see ADR-0002).
- **FR-3 — Playlists.** Conductor MUST transfer the user's **owned playlists** (name, and ordered
  track membership), subject to capability gating (FR-7). Playlists the user only follows, and
  collaborative playlists, are **out of scope** for v1.
- **FR-4 — Saved albums.** Conductor MUST transfer the user's **saved albums**, subject to
  capability gating (FR-7).
- **FR-5 — Saved songs.** Conductor MUST transfer the user's **saved songs** — the service's native
  saved/liked-tracks collection, modeled as its own item type and never as a playlist — subject to
  capability gating (FR-7).
- **FR-6 — Saved artists.** Conductor MUST transfer the user's **saved/followed artists**, subject
  to capability gating (FR-7).
- **FR-7 — Capability gating.** For each library item type, Conductor MUST attempt the transfer
  **only when both** the source and destination declare support for that type (see §6). Some
  services lack saved artists or saved songs; Conductor MUST degrade gracefully — skipping the
  unsupported type and reporting it (see §8) — rather than erroring out the whole transfer.
- **FR-8 — Dry run.** Conductor MUST support a **dry-run** mode that performs every read and match
  step and produces the reports (§8) but writes nothing to the destination.

---

## 5. Configuration Requirements

- **FR-9 — Config-driven.** A transfer MUST be defined by a user-provided **YAML** configuration
  file that specifies the services involved, the source → destination direction, and per-service
  credential references. A single config file describes **exactly one** transfer in v1.
- **FR-10 — Credentials via environment.** Credentials MUST be referenced indirectly via
  **environment variables** (or equivalent secret references), never committed inline in the config
  file.
- **FR-11 — Selectable item types.** The config MUST allow the user to select which library item
  types to transfer. When unspecified, the default is **all mutually supported types** (per FR-7).
- **FR-12 — Configurable match confidence.** The config MUST allow the user to set the confidence
  threshold (FR-16) **per library item type**, with a single global default applied where no
  per-type override is given.
- **FR-13 — Configurable idempotency mechanism.** The config MUST allow the user to select the
  idempotency mechanism (FR-21). The default is the local ledger; reading destination state and
  diffing is a later-version alternative (see ADR-0003).

Illustrative configuration (shape is illustrative, not normative):

```yaml
# conductor.yaml
services:
  spotify:
    # authentication method is service-specific and TBD (see ADR-0002)
    credentials_env: SPOTIFY_CREDENTIALS
  youtube_music:
    credentials_env: YTMUSIC_CREDENTIALS

transfer:
  source: spotify
  destination: youtube_music
  item_types:            # optional; defaults to all mutually supported types
    - playlists
    - saved_albums
    - saved_songs
    - saved_artists
```

---

## 6. Service Capability Model

Every supported service **MUST declare** which of the four library item types it supports. A
transfer applies FR-7 (capability gating) using these declarations.

**Capability matrix** (initial targets; entries marked *TBV* are **to be verified at
service-integration time** against each service's current API — consistent with deferring auth
specifics, capability facts are confirmed against the live API when the connector is built, not
guessed now):

| Service       | Playlists | Saved albums | Saved songs | Saved artists |
| ------------- | :-------: | :----------: | :---------: | :-----------: |
| Spotify       |    ✅     |     ✅       |    ✅       |     ✅        |
| YouTube Music |    ✅     |    ✅ *TBV*   |   ✅ *TBV*  |    ✅ *TBV*    |

- **FR-14 — Capability declaration.** Each service MUST declare its supported item types, and
  Conductor MUST honor those declarations when planning a transfer.
- **FR-15 — Unsupported types reported.** When an item type is unsupported by either endpoint,
  Conductor MUST skip it and surface it in the report as *"skipped — unsupported by `<service>`"*
  (see §8). This section is the registry to update when adding a service (see §10).

---

## 7. Matching Requirements

Matching is fully automated and matches by durable content attributes rather than service-specific
IDs; see **ADR-0001** for the decision and its rationale.

- **FR-16 — Match by durable attributes with a confidence threshold.** Conductor MUST match items
  across services using durable content attributes — for example **ISRC** for tracks, **UPC**
  and/or normalized name+artist for albums, and normalized name for artists and playlists — and
  MUST accept a candidate as a match only when its confidence is at or above the configured
  threshold (FR-12).
- **FR-17 — No confident match is a distinct outcome.** Conductor MUST treat **"no confident match
  found"** as a distinct, reportable outcome (see §8) — never a silent drop, never a guess, and
  never a crash. There is no interactive disambiguation in v1.
- **FR-18 — Partial collections transfer the matched subset.** When some items in a playlist, the
  saved-songs set, or a saved album match confidently and some do not, Conductor MUST transfer the
  matched subset and report each unmatched item individually (FR-19) rather than failing the whole
  collection.

---

## 8. Reporting & Failure Handling Requirements

- **FR-19 — Detailed report.** On completion, Conductor MUST produce a **detailed report** of what
  transferred successfully and what did not, itemized with a reason for each non-success. The report
  MUST be produced in two forms: a **human-readable HTML** summary and a **machine-readable JSON**
  record. (Serving the HTML from a local server is a later-version enhancement.)
- **FR-20 — Failure taxonomy.** Each non-success MUST be classified as one of:
  - **Not available on destination** — the artist/song/album/playlist does not exist on the
    destination platform. Listed in the report with the item and reason.
  - **No confident match** — a candidate could not be matched with sufficient confidence (FR-17).
    Reported **distinctly** from "not available".
  - **Unsupported item type** — skipped due to capability gating (FR-15).
  - **Transient / networking error** — a retryable failure (timeouts, rate limits, 5xx, connection
    errors).
- **FR-21 — Idempotent re-runs via a ledger.** Re-running a transfer (including from the retry
  store) MUST be idempotent — it MUST NOT duplicate items that were already transferred
  successfully. The default mechanism is a local **ledger**; because appending a track to a playlist
  is the one operation that is not naturally idempotent, Conductor MUST read the destination
  playlist's current membership before appending, regardless of the configured mechanism (see
  ADR-0003, ADR-0004).
- **FR-22 — Separate retry store.** Items that failed due to transient/networking errors MUST be
  persisted to a **separate retry store** that the user can re-run at their discretion. These MUST
  NOT be mixed into the permanent-failure listings in the report.
- **FR-23 — Atomic artifacts.** The ledger, retry store, and reports are computed during a run and
  written only on **completion**. A run interrupted mid-execution MUST leave no ledger, no report,
  and no retry store (see ADR-0003).
- **FR-24 — Local artifact layout.** All local artifacts for a transfer (ledger, retry store, JSON
  report, HTML report) MUST be written under a per-transfer directory
  (`.conductor/<source>-<destination>/`), so that re-runs deterministically find prior state.

---

## 9. Non-Functional Requirements

- **NFR-1 — Privacy.** Conductor MUST NOT emit telemetry. Library data and credentials MUST NOT
  leave the user's machine except in calls to the user's own service APIs.
- **NFR-2 — Security.** Credentials MUST be sourced from environment/secret references (FR-10), MUST
  NOT be logged, and MUST be redacted from the report and any diagnostic output.
- **NFR-3 — Safety.** Transfers MUST be **additive** by default — Conductor MUST NOT delete or
  modify existing content on the source or destination. Combined with FR-21, operations are safe to
  re-run.
- **NFR-4 — Usability.** A transfer MUST be drivable from a single config file, with clear CLI
  feedback during the run and an actionable report at the end.
- **NFR-5 — Extensibility.** Adding a service MUST require only declaring its capabilities (§6) and
  implementing the required per-item-type operations. Concrete design of that extension mechanism is
  deferred to a later document.

---

## 10. Supported Services

Currently targeted services:

- **Spotify**
- **YouTube Music**

When adding a new service, update the **capability matrix** in §6 with the service's supported item
types (verifying each against the service's current API), and ensure it satisfies the credential
(FR-2, FR-10) and matching (§7) requirements.
