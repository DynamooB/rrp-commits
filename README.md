# rrp-commits — public witness log

This repository is a **timestamped witness log**. It exists to prove that a
resolution-risk score for a prediction market was produced *before* that market
resolved — and that the score was never edited afterwards.

It contains exactly two things: this README and [`commits.log`](commits.log).
No code, no scores, and no pre-resolution reports are ever published here.

## What is a resolution-risk score?

It scores the risk that a prediction market **resolves badly** — ambiguously,
through a dispute, or contrary to what its own rules appear to promise. It is
not a forecast of the underlying event. A market can be a near-certainty and
still carry severe resolution risk if the rules leave the resolver too much
latitude.

## Why a log like this has to exist

Any track record published *after* outcomes are known is unfalsifiable. Nothing
stops someone from writing down a call once the answer is in, or quietly
revising a bad one. The only fix is to publish something binding, in public,
before the outcome exists — and to make it verifiable by a stranger who assumes
bad faith.

A hash is the smallest such thing. It reveals nothing about the score (so the
call isn't leaked before resolution) while making the score impossible to
change (so the call can't be revised after).

## The scheme

**At scoring time, before the market resolves:**

1. A full report is assembled: the market snapshot as of scoring, every scoring
   run, the composite, and the engine version that produced it.
2. That report is canonicalised as sorted-key, UTF-8 JSON with compact
   separators, then hashed with SHA-256.
3. One line is appended to `commits.log` and pushed here:

   ```
   {market_id},{sha256_of_report},{utc_timestamp}
   ```

   **GitHub's commit timestamp is the witness.** The author timestamp in a git
   commit can be forged; the moment GitHub received the push cannot be, and it
   is visible in this repository's commit history.

**After the market resolves:**

4. The full report JSON is published. Anyone can canonicalise it, hash it, and
   check the result against the line that was already public — and check, via
   this repo's history, that the line predates the resolution.

## Verifying a claim yourself

Given a published report file:

```bash
python -c "import json,hashlib,sys; p=json.load(open(sys.argv[1])); print(hashlib.sha256(json.dumps(p,sort_keys=True,ensure_ascii=False,separators=(',',':')).encode()).hexdigest())" report.json
```

Then:

- Confirm that hash appears in `commits.log`.
- Run `git log -S"<the hash>" --format="%H %cI"` in a clone of this repo to find
  the commit that introduced it, and check its **committer** date.
- Confirm that date precedes the market's resolution.

If the hash isn't in the log, or its commit postdates resolution, the claim is
worthless. That is the intended failure mode: the scheme is designed so you do
not have to take anyone's word for anything.

## What this log does *not* prove

- It does not prove a score was *good* — only that it was fixed in advance.
- It does not prove the report was scored on an honest snapshot. The report
  binds the market snapshot by hash, but the snapshot itself came from the
  Polymarket Gamma API at fetch time.
- It does not prevent selective disclosure: someone could commit many scores
  and reveal only the flattering ones. Judge a track record by the *ratio* of
  published reports to committed lines — every line here should eventually get
  a report.
