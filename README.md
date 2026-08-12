# list-hygiene-judge

Public source and provenance for the runx skill **`list-hygiene-judge`**: a graph
runner that decides whether a contact should be verified, suppressed, or
re-permissioned from read engagement and bounce evidence, records the consent
transition as one compare-and-set `append_event` through `data-store`, and never
sends. `send-as` is the downstream enforcer — a separate governed run that reads
the recorded state at send time and gates delivery.

- Package: `skills/list-hygiene-judge/`
- Harness: `runx harness ./skills/list-hygiene-judge` — three cases, green, and
  green on repeat runs against a warm store.
- Upstream PR: https://github.com/runxhq/runx/pull/408

See `skills/list-hygiene-judge/SKILL.md` for the judgment order and the input
contract.
