# TAPE-WITNESS — anonymous click-anomaly detector

Prove what your own screen recorded vs what the public record claims, at the
minutes YOU acted. No strategy, no predictions, no identifying data. The tool
is a ruler, not a trader.

## What this is, in plain words

You are trading. Your screen shows you a chart — candles and volume bars. There is also a public record of that same market: the official data everyone else sees, and the data your own account later exports.

**On a quiet minute, the two agree.** Nobody is trading, nothing is happening, and what your screen showed is exactly what the public record shows. This tool has checked thousands of those quiet minutes. They match.

**At the minute you click, they stop agreeing.**

- Your screen showed a small amount of volume. The public record shows **double**.
- You sold. The public record prints your sell as **buying**.
- Your screen showed the price reaching one high. The public record claims a **higher high your screen never showed**.
- And sometimes, when you pull the same data again later, **the minute is simply gone** — it was in the early export, it is not in the later one.

This does not happen in the quiet minutes around your trade. It happens at **your** minutes — the moments of a live trade, when a real person acts. Thousands of quiet minutes: no anomaly. Action minutes: the anomaly, again and again.

**Nobody knows why.** This tool does not claim a cause, and it does not name a culprit. What it proves is narrower, and harder to argue with: **the mismatch is conditional on action.** It is off when you are still, and on when you act. That on/off pattern is a fact anyone can measure — and that is the invitation. **Test it yourself.** Record your screen, pull your data twice, run the ruler. If it happens on your account too, the pattern is bigger than one person's screen. If it doesn't, the census will show that honestly too.

Why it matters, in one sentence: **if the public is shown one set of candles and volumes, and the person actually trading is shown another set at the very moment of the trade, then the data is misleading — and until now it was unprovable.** A screen recording plus two data pulls makes it provable, recorded, and repeatable by anyone.

*The anomaly in one picture — quiet minute vs. action minute vs. the later pull — is being produced as a plate for this page (docs/anomaly-plate.png) and will be posted here the moment it is ready.*

## The anomaly patterns it grades

On a dead, boring, barely-moving asset — overnight micro futures, median
volume single digits per minute — the following cluster ONLY at the
operator's own action minutes, never in the surrounding quiet hours:

1. `inverted_classification` — market sells printing as up-volume
   (UpVol > 0, DnVol = 0 at proven sell minutes).
2. `volume_inflation` — public volume 2x+ the locally recorded screen
   volume at the same minute.
3. `volume_spike` / `spike_unclassified` — 5x+ quiet-median volume bars,
   with classification columns zeroed (UpVol 0 / DnVol 0) at execution
   minutes.
4. `reclick_signature` — paired re-clicks inside a short window (default
   30s), size change recorded but not required for the pairing.
5. `removal_proof` / `minute_absent` — bars present in early data pulls,
   absent from later pulls of the same feed (exports compared in ARGUMENT
   order, not filename order).
6. `screen_price_divergence` — the public print claims highs beyond the
   screen's recorded range by a configurable threshold. Direction is
   classified only when the action side is known.
7. `export_behavior_anomaly` — repeated empty payloads and/or forced
   logouts in the account's own export attempts (observed sequence only;
   no causal claim).

None of these alone proves intent. The tool's claim is narrower and harder:
the anomaly is CONDITIONAL ON ACTION. Quiet minutes — thousands of them —
show none of it. That asymmetry is measurable, reproducible, and it is the
whole point.

## What you need

- A continuous screen recording of your trading session (OBS or equivalent)
- Data exports of the same instrument pulled at multiple times (early pull
  = proof of existence; later pull = proof of absence)
- This repo's tools

## The pipeline

```
# 1. Extract timestamped frames from your recording
python3 witness/frames.py session.mkv --out frames/ --every 20 \
    --start-time 2026-01-05T01:58:43

# 2. Log your executions (from broker toasts visible in the recording)
#    clicks.txt: one per line  "2026-01-05T01:59:52 SELL 1 @ 105.37"
python3 witness/ledger.py frames/ clicks.txt --start 01:58:43 --interval 20 \
    --out ledger.json

# 3. Reconcile against the public export(s) — ARGUMENT ORDER = pull order
python3 witness/reconcile.py ledger.json export_early.csv export_late.csv \
    --out diff.json --export-attempts attempts.jsonl

# 4. Grade the anomaly (baseline = the most complete late export)
python3 witness/grade.py diff.json --export export_late.csv --out verdict.json
```

Every step appends a JSONL receipt — never overwritten:
- frames.py: recording SHA-256, extraction parameters, per-frame hashes
- ledger.py: invocation receipt (clicks hash, frames.jsonl hash, every
  referenced frame's hash, --start/--interval, output hash) + one receipt
  per action
- reconcile.py: hashes of ledger, every export, and the export-attempts
  file; the diff itself carries a self-contained `sources` section
- grade.py: hashes of the diff and the baseline export

The hash manifest (`python3 witness/hash.py .`) is an append-only
OBSERVATION HISTORY: a changed file gets a new dated line, so tampering
and edits are both visible. It is a LOCAL observation log, not proof the
log itself was not altered — for stronger custody, copy or sign the final
manifest to an independent location.

Receipts and HASHES.txt are deliberately gitignored so private evidence
never lands in a public repo by accident. To archive a session's custody
records, bundle them yourself: the receipts, HASHES.txt, frames.jsonl,
ledger, diff, and verdict.

## The community census — why this exists

One operator's receipts are an anecdote. A thousand operators' receipts,
graded with the same ruler, are a census. TAPE-WITNESS is built so the
community can answer the population question itself:

    does the action-conditional anomaly reproduce across accounts,
    brokers, assets, and time — or is it one screen's story?

How it works, end to end:

1. You grade YOUR OWN session with the pipeline above. Your raw data
   never leaves your machine.
2. `witness/review.py submit verdict.json --asset micro-futures \
      --window "2026-01-05T01:30/04:30" --out my_submission.json`
   strips the verdict down to structured COUNTS only — no account
   numbers, no names, no paths, no raw records. The sanitizer REJECTS
   any submission carrying identity fields or raw data.
3. Anyone can aggregate: `witness/review.py census submissions/*.json`
   produces the population report: flagged-session rate with Wilson 95%
   bounds, stratified by asset class, with population-wide pattern
   totals.

The census never aggregates raw data — only already-anonymized counts.
It makes no causal claims and names no platforms. It answers exactly one
question: does the pattern reproduce? If it does, at scale, across
independent operators who never shared a byte of raw data — that is a
fact worth knowing. If it doesn't, that is worth knowing too, and the
census will show it honestly.

## Tests

```
python3 -m unittest discover -s tests -v
```

12 tests lock down reconciliation (argument-order removal proof, minute
aggregation, no-false-positive screen matching), grading (structured
pattern IDs, actions vs minutes, unsupported patterns never counted),
hashing (idempotent, changed-file observations, correct SHA-256), and the
full E2E pipeline against `examples/`. CI runs them on every push.

## Anonymity law (read before publishing anything)

- Screenshots you publish must show ONLY the plain trading chart of the
  public platform. NEVER your prediction layers, overlays, drawings, or
  any second chart — your method is your own.
- Strip metadata from every image and video you share.
- Refer to the operator in third person ("the operator", "user A").
- The example data in `examples/` is fully synthetic — numbers that
  reproduce the pattern shapes without being anyone's real session.

## What this repo is NOT

- Not a trading strategy, signal service, or prediction tool
- Not an accusation against any named platform — it is a measurement tool
  and a reproducible method
- Not legal advice

## License

MIT. Measure everything. Keep your own records append-only. They can erase
their logs; they cannot erase yours.

## The witness family
- [Witness Stand](https://github.com/amartieai/witness-stand) — a judge for the judges
- [Trade Witness](https://github.com/amartieai/trade-witness) — the always-on trading witness
- The stand itself: [r/TheWitnessStand](https://www.reddit.com/r/TheWitnessStand/)
- Home: [amartie.org](https://amartie.org)
