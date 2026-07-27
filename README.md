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

## Disclosure policy: no cherry-picking

**Every hash committed to this log gets its full report published when the
market resolves — hit or miss.** A score that turns out to be embarrassing is
published on the same terms as one that looks prescient. Committing a hash is a
standing obligation to reveal, not an option to reveal if the result flatters.

This is the policy that makes the rest of the scheme mean anything. Hashes are
cheap: anyone can commit a hundred calls, reveal the six that landed, and let
the rest rot unrevealed. That produces a track record with an unbounded and
invisible denominator, which is worse than no track record at all, because it
looks rigorous.

So the accounting rule is deliberately unforgiving: **any hash whose market has
resolved without a corresponding published report should be counted as a miss.**
Not as pending, not as withdrawn — a miss. A reader auditing this log is
entitled to treat silence as failure, and the ratio of revealed reports to
resolved commitments is the honest measure of the record.

### Re-scoring and which commitment binds

A market can legitimately be scored more than once — the engine improves, a
market's rules get clarified, a run turns out to be unusually noisy. Each
re-score appends a new hash, so a single market may hold several lines here.

**The last hash committed before the market resolves is the binding call for
track-record purposes.** That is the one the record is scored against, win or
lose. Choosing it is not a matter of preference: it is fixed by the resolution
timestamp, which nobody controls, so the binding call is always identifiable
after the fact and can never be selected retroactively to flatter the record.

Earlier hashes for the same market are **not** discarded. Each is revealed at
resolution alongside the binding one, marked as superseded and carrying the
reason it was re-scored. The report format records this directly: every report
carries a `supersedes` field naming the hash it replaces and a `rescore_reason`
field explaining why, so the chain is machine-checkable rather than a claim in
prose. A superseded report with a missing or evasive reason should be read as
what it is — an attempt to quietly bury an earlier call.

The obvious abuse this invites is re-scoring a market repeatedly until one
version looks good, then leaning on that one. The accounting rule above already
forecloses it: every hash in the chain is revealed, so a market with six
commitments and five superseding reasons is visibly a market that was scored
six times, and a reader can judge it accordingly.

## What this log does *not* prove

- It does not prove a score was *good* — only that it was fixed in advance.
- It does not prove the report was scored on an honest snapshot. The report
  binds the market snapshot by hash, but the snapshot itself came from the
  Polymarket Gamma API at fetch time.
- It cannot *enforce* the disclosure policy above — only make violations
  visible. Nothing in cryptography compels a reveal; what the log guarantees is
  that an unrevealed commitment stays permanently on the record, countable by
  anyone. Enforcement is the reader's, by applying the accounting rule.
