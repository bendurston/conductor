# Conductor

**Safely move your music library between streaming services.**

Conductor is a command-line tool that copies your music library from one streaming service to
another — for example, from Spotify to YouTube Music. It transfers your **playlists**, **saved
albums**, **saved songs**, and **saved artists**, using **your own** API credentials for each
service.

## Why Conductor

- **Privacy-first** — it runs on your machine, and nothing leaves it except calls to your own
  service APIs. No hosted service, no central storage of your credentials or library.
- **Safe** — transfers are additive. Conductor never deletes or modifies content on either service,
  and re-running a transfer won't create duplicates.
- **Config-driven** — one YAML config file describes what to transfer and where.
- **Easy to use** — run it, then read a clear report of what moved and what didn't.

## Supported services

- **Spotify**
- **YouTube Music**

Not every service supports every kind of saved content, so Conductor only transfers item types that
both the source and destination support. See the capability matrix in
[`docs/requirements.md`](docs/requirements.md#6-service-capability-model) for details.

## Getting started

1. **Install** Conductor.
2. **Create a config file** (YAML) naming your source and destination services, and referencing
   your API credentials via environment variables.
3. **Run the transfer.**
4. **Read the report** of what transferred and what didn't.

```yaml
# conductor.yaml
services:
  spotify:
    credentials_env: SPOTIFY_CREDENTIALS
  youtube_music:
    credentials_env: YTMUSIC_CREDENTIALS

transfer:
  source: spotify
  destination: youtube_music
```

## Reports & retries

When a transfer finishes, Conductor gives you a detailed report of everything that moved and, for
anything that didn't, the reason why (for example, a song that isn't available on the destination).
Items that failed because of temporary network problems are saved separately so you can retry them
whenever you like.

## Documentation

- **[`docs/requirements.md`](docs/requirements.md)** — the full, detailed specification of how
  Conductor behaves; the authoritative, agent-facing source of truth.
- **[`CONTEXT.md`](CONTEXT.md)** — the project glossary: the canonical vocabulary the docs and code
  use.
- **[`docs/adr/`](docs/adr/)** — architecture decision records: the hard-to-reverse choices and why
  they were made.
