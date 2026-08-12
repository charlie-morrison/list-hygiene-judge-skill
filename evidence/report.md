# list-hygiene-judge — delivery report (bounty #68)

**Package** `charlie-morrison/list-hygiene-judge@sha-db5cd96bfb28` · trust tier `community`
**Registry page** https://runx.ai/x/charlie-morrison/list-hygiene-judge
**Source** https://github.com/charlie-morrison/list-hygiene-judge-skill
**PR** https://github.com/runxhq/runx/pull/408

Toolchain: `runx --version` → `runx-cli 0.8.2`. Publish, install, dogfood and
verify were all run with that binary.

## Delivery at a glance

- **Package** — `charlie-morrison/list-hygiene-judge@sha-db5cd96bfb28`, trust tier `community`, live at https://runx.ai/x/charlie-morrison/list-hygiene-judge@sha-db5cd96bfb28
- **Toolchain** — `runx --version` → `runx-cli 0.8.2`; publish, install, dogfood and verify were all run with that binary (bounty floor is 0.6.14)
- **Publish** — `runx login --provider github --for publish --from-gh`, then `runx registry publish ./skills/list-hygiene-judge/SKILL.md --registry https://api.runx.ai` → published, target hosted
- **Local harness before publish** — `runx harness ./skills/list-hygiene-judge` green on all three cases, and green again on repeat runs against a warm store
- **Hosted harness after publish** — `3 fixtures passed for list-hygiene-judge`, run id `runx-harness:charlie-morrison/list-hygiene-judge:sha-db5cd96bfb28`
- **Clean install** — `runx add charlie-morrison/list-hygiene-judge@sha-db5cd96bfb28 --registry https://api.runx.ai` succeeds into an empty directory
- **Dogfood** — post-publish run of the published package; all four steps succeeded (`read-contact`, `decide`, `append-transition`, `readback`) and sealed `sha256:5429ba26c960c52cba70563fce5af63f56bbb96c69f2837e79222ec097d12a02`
- **Verdict recorded** — `suppress`, because `hard_bounces=3` was read from `contacts/contact:dogfood:c-9002`; transition written at `new_version: 1` bound to `idempotency_key=contact:dogfood:c-9002:hygiene:v1`, confirmed by the read-back
- **Verification** — `runx verify` returns `valid: true` with digest and content-address both valid; signature mode is `local-development` because the receipt issuer is the local runtime skeleton, and that is stated rather than dressed up
- **Refusals exercised** — suppression without bounce evidence, re-permission over an active unsubscribe marker, ambiguous bounce recovery, and a stale `expected_version` each refuse to write; the last two escalate to a human approval lane
- **No send** — `send_effect: none`; `send-as` is the downstream enforcer that reads the recorded state at send time and gates delivery
- **PR** — https://github.com/runxhq/runx/pull/408, head `3748ca9bfa540de3b752c5e93e8be8a7fff04c8f`, containing `skills/list-hygiene-judge/` with `X.yaml`, `SKILL.md`, fixtures and captured harness evidence

## What the skill is

List hygiene is the judgment between engagement decay and suppression, and the
dangerous part is the durable consent-state transition. The graph reads a contact
through the data-store keyed by the contact as the domain entity, decides whether
to verify, suppress, or re-permission, and records the transition by appending
exactly one event to that contact's stream under `idempotency_key` +
`expected_version` compare-and-set.

**It never sends.** `send-as` is the downstream enforcer: a separate governed run,
dispatched by naming, that reads the recorded state at send time and gates
delivery. A suppressed contact cannot receive a campaign because of what is
recorded here, not because of anything this skill emits.

## The judgment

Evaluated in this order, and every branch is grounded in state that was read:

1. **Evidence gate.** A missing `opens_count`, `clicks_count`, `hard_bounces` or
   `recency_days` counter is a stop, not a zero. An unreadable contact projection
   is a stop. A missing `bounce_policy` is a stop.
2. **Ambiguous bounce recovery.** The contact hard-bounced *and* has since engaged
   inside the decay window. Suppressing would drop a live human; re-permissioning
   would ignore a real delivery failure. Neither is decidable from the evidence,
   so neither is taken — it escalates to a human approval lane.
3. **Suppress**, only on `hard_bounces > 0` read from the store. This is the sole
   path to suppression: the judgment refuses to suppress without bounce evidence.
4. **Active unsubscribe marker** — refuses to re-permission over one, escalates.
5. **Re-permission** — `recency_days` past `decay_threshold_days`, no marker, no
   bounces.
6. Otherwise `no_change`, and no event is appended.

The append and the read-back are both guarded on `decision.writes == true`, so
every stop path provably emits no append.

## The version gate, and why it is not just a staleness check

A stream sitting exactly one event ahead of what the caller read, whose newest
event is the very transition this judgment would record, is a **retry of our own
write** — not stale state. The store's append is keyed by `idempotency_key` and
returns the recorded version instead of double-applying, so the honest response is
to stand by the same decision and let the append replay. Any other version drift
is genuine staleness: the read is behind the stream, and an append would apply
against state that was never read.

This was found by testing rather than by reasoning. The first version of the
harness was green exactly once: the second run hit the persisted store, the
projection was one version ahead, and the stale-version check correctly refused —
which meant a reviewer re-running the harness would have seen it red. Two things
were wrong and both are fixed: the version gate did not distinguish a replay from
a conflict, and the appended event embedded the prior-projection snapshot, whose
version moves, so a retry landed as an idempotency **conflict** instead of a
replay. The event now carries the transition and the evidence it rested on, and
nothing that mutates between attempts.

## Harness

`runx harness ./skills/list-hygiene-judge` — three inline cases on the default
runner:

| case | outcome |
|---|---|
| `sealed_decay_re_permission` | sealed — 400 days idle over a 180-day threshold, no marker, no bounces → `re_permission`, one append |
| `sealed_hard_bounce_suppress` | sealed — `hard_bounces: 2` → `suppress`, one append |
| `stop_missing_or_stale_evidence` | refused — `engagement_history` absent → stop, both guards block, no append |

Green locally before publish, and green on the hosted registry harness after
publish (`3 fixtures passed`, run id
`runx-harness:charlie-morrison/list-hygiene-judge:sha-db5cd96bfb28`). Green on
repeat runs against a warm store.

Beyond the three declared cases, each decision branch was exercised directly
against the runner: fresh write, idempotent replay of our own transition, a
concurrent writer's event at the same version delta (stops), a version far ahead
(stops), ambiguous bounce recovery (escalates, no append), and an active
unsubscribe marker (escalates, no append).

## Dogfood

Post-publish run of the published package, not a harness fixture seal:

```
runx add charlie-morrison/list-hygiene-judge@sha-db5cd96bfb28 --registry https://api.runx.ai
runx skill charlie-morrison/list-hygiene-judge@sha-db5cd96bfb28 --registry https://api.runx.ai --json \
  -i data_source_ref=local://charlie-morrison-list-hygiene/dogfood \
  -i store_id=list-hygiene-judge-dogfood-v1 -i resource=contacts \
  -i aggregate_id=contact:dogfood:c-9002 -i expected_version=0 \
  -i idempotency_key=contact:dogfood:c-9002:hygiene:v1 \
  --input-json engagement_history='{"opens_count":0,"clicks_count":0,"hard_bounces":3,"recency_days":45}' \
  --input-json bounce_policy='{"hard_bounce_action":"suppress","decay_threshold_days":180}' \
  -i current_consent_state=subscribed
```

All four steps succeeded — `read-contact` (`data.read_projection`), `decide`,
`append-transition` (`data.append_event`), `readback` — and the run sealed as
`sha256:5429ba26c960c52cba70563fce5af63f56bbb96c69f2837e79222ec097d12a02`.

Verdict: `suppress`, because `hard_bounces=3` was read from
`contacts/contact:dogfood:c-9002`, with `bounce_policy.hard_bounce_action=suppress`.
The transition was recorded at `new_version: 1` bound to
`idempotency_key=contact:dogfood:c-9002:hygiene:v1`, and the read-back confirms it
on the contact's projection. No send was performed; `send_effect` is `none`.

## Verification, stated plainly

```
runx verify --receipt <receipt.json> --allow-local-development-signatures --json
→ valid: true · digest: valid · content_address: valid · signature: local-development, valid
```

The receipt issuer is the local runtime skeleton (`issuer.type=local`,
`kid=runtime-skeleton`), so `runx verify` reports signature mode
`local-development` and needs that flag. The digest and content-address checks are
unconditional and both passed; the signature check is only as strong as that local
issuer, and I am not going to describe it as more than it is. Lineage is reported
`unverified` because a single receipt cannot prove a receipt tree.

## Note on the package shape

The graph composes the native `data.read_projection` and `data.append_event` tools
directly, which is what `runx/data-store` does internally. An earlier revision
bundled a local JSON event-store adapter instead; it passed every local harness
and failed the hosted publish harness, which is the one that matters for anyone
installing this from the registry.

`CONTRIBUTING.md` points community skills at standalone packages and reserves the
runx repo for the first-party lane. The PR is opened against `runxhq/runx` because
this bounty requires the package files to land there; the standalone package is at
the source repo above either way.
