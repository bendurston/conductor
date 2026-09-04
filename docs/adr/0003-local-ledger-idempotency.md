# Idempotency via a local ledger, computed in memory and flushed on completion

Conductor makes re-runs idempotent with a local Ledger of what has already been transferred. The
mechanism is config-selectable, and the default is the local Ledger; the alternative — reading
Destination state and diffing on every run — is more robust to the user editing the Destination
between runs, but Conductor is close to a single-use tool, so that cost is not worth paying by
default and it is deferred to a later version.

The Ledger is held in memory during a run and flushed to disk, together with the report and Retry
store, only when the run completes. A run interrupted mid-execution therefore leaves no Ledger, no
report, and no Retry store — artifacts are all-or-nothing. The consequence for idempotency is
handled narrowly in ADR-0004.
