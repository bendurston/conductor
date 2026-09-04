# Check playlist membership on the destination before appending a track

Adding a Saved song, Saved album, or Saved artist to a Destination is naturally idempotent — the
library is a set, so re-adding is a no-op. Appending a track to a Playlist is not: a repeated
append can create a duplicate. Because the default idempotency mechanism is a local Ledger
(ADR-0003) and an interrupted run leaves no Ledger, a re-run after an interruption could otherwise
duplicate tracks in a Playlist and violate the "must not duplicate" requirement.

So, as a deliberate and narrow exception to the Ledger-only default, playlist append always reads
the current Destination playlist membership before adding a track, regardless of the configured
mechanism. This is the one operation that is not naturally idempotent; the set-like item types need
no such check. Do not remove this read on the assumption it is redundant with the Ledger — it
exists precisely for the interrupted-then-rerun case the Ledger cannot cover.
