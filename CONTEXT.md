# Conductor

Conductor is a command-line tool that copies a user's music library from one music streaming
service to another. This glossary fixes the vocabulary the project uses to talk about that
transfer. It is a glossary only — no implementation detail, no requirements. Requirements live in
[`docs/requirements.md`](./docs/requirements.md); decisions live in [`docs/adr/`](./docs/adr/).

## Language

### Services and transfers

**Service**:
A supported music streaming platform, such as Spotify or YouTube Music.
_Avoid_: Provider, platform, backend.

**Source**:
The Service a Transfer reads a library from.
_Avoid_: Origin, from-service.

**Destination**:
The Service a Transfer writes a library to.
_Avoid_: Target, to-service, sink.

**Transfer**:
A single, user-initiated, one-directional copy of selected Library item types from a Source to a
Destination. A Transfer is additive: it never deletes or modifies existing content.
_Avoid_: Sync, migration, export, copy job.

**Dry run**:
A Transfer that performs every read and Match step and produces the reports, but writes nothing to
the Destination.
_Avoid_: Preview, simulation, test run.

**Capability**:
Whether a given Service supports a given Library item type. A Transfer attempts an item type only
when both the Source and Destination declare the Capability for it.
_Avoid_: Feature, support flag.

### Library content

**Library item type**:
One of the four categories of saved content Conductor transfers: Playlists, Saved albums, Saved
songs, and Saved artists.
_Avoid_: Content type, entity, resource.

**Playlist**:
A user-owned, named, ordered collection of tracks. Only playlists the user owns are in scope;
playlists the user merely follows, and collaborative playlists, are not.
_Avoid_: List, mix.

**Saved songs**:
The Service's native collection of individually saved tracks — what Spotify calls "Liked Songs".
This is a distinct Library item type, never modeled as a Playlist.
_Avoid_: Liked songs, favorites, saved tracks (as separate concepts).

**Saved album**:
An album the user has saved to their library.
_Avoid_: Favorited album, bookmarked album.

**Saved artist**:
An artist the user has saved or followed — the two are the same concept here.
_Avoid_: Followed artist (as a separate concept), subscribed artist.

### Matching and outcomes

**Match**:
The mapping of a Source item to the equivalent item on the Destination, established from durable
content attributes (such as ISRC for tracks) rather than Service-specific IDs. A Match carries a
confidence; a Match below the Confidence threshold is not a Match but a distinct "no confident
match" outcome.
_Avoid_: Lookup, resolution, mapping.

**Confidence threshold**:
The minimum confidence at or above which a candidate is accepted as a Match. Configurable
per Library item type, with a single global default.
_Avoid_: Cutoff, score limit.

**Ledger**:
The record of which Source items a Transfer has already copied to the Destination, used to make
re-runs idempotent.
_Avoid_: Journal, log, history.

**Retry store**:
The set of items that failed a Transfer for transient reasons (timeouts, rate limits, 5xx,
connection errors) and can be re-run at the user's discretion. Kept separate from permanent-failure
outcomes.
_Avoid_: Retry queue, dead-letter queue, failure log.
