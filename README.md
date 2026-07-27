# rrp-commits — public witness log

This repository is a **timestamped witness log**. It exists to prove that a
resolution-risk score for a prediction market was produced *before* that market
resolved — and that the score was never edited afterwards.

It contains this README, [`commits.log`](commits.log), and — for entries before
the chain-witness upgrade — [`batch_anchor.json`](batch_anchor.json).
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

## The scheme (chain-witnessed, all entries from the batch anchor block onward)

**At scoring time, before the market resolves:**

1. A full report is assembled: the market snapshot as of scoring, every scoring
   run, the composite, and the engine version that produced it.
2. That report is canonicalised as sorted-key, UTF-8 JSON with compact
   separators, then hashed with SHA-256.
3. A **zero-value transaction is sent on Base mainnet** with the digest in
   calldata (prefix `rrp:` followed by the 32 raw digest bytes). The on-chain
   timestamp is the primary witness — it is set by the network and cannot be
   forged by the author.
4. One line is appended to `commits.log` and pushed here (5-column format):

   ```
   {market_id},{sha256_of_report},{utc_timestamp},{base_tx_hash},{block_number}
   ```

   The GitHub commit timestamp provides a second, human-readable timestamp.

**After the market resolves:**

5. The full report JSON is published. Anyone can canonicalise it, sha256 it,
   and confirm the digest matches the calldata of the referenced transaction.

## Log line formats

**Entries from block N onward (per-commit chain anchor, 5 columns):**
```
{market_id},{sha256},{utc_ts},{0x_tx_hash},{block_number}
```

**Entries before block N (3 columns, batch-anchored retroactively):**
```
{market_id},{sha256},{utc_ts}
```

These 3-column entries are covered by `batch_anchor.json` — a single Base
transaction carrying the Merkle root of all of them, with individual Merkle
proofs included in the JSON file.

## Verifying a per-commit entry (5-column)

Given a published report file and a log line:

```python
import json, hashlib

# 1. Recompute the digest
report = json.load(open("report.json"))
canonical = json.dumps(report, sort_keys=True, ensure_ascii=False, separators=(",", ":"))
digest = hashlib.sha256(canonical.encode()).hexdigest()
print("digest:", digest)

# 2. Compare against the log line's second field — they must match.
```

```python
# 3. Fetch the Base transaction and verify calldata
from web3 import Web3
w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))
tx = w3.eth.get_transaction("0x<tx_hash_from_log_line>")
expected = b"rrp:" + bytes.fromhex(digest)
assert bytes(tx["input"]) == expected, "calldata mismatch"
print("chain anchor verified at block", tx["blockNumber"])
```

Or use the CLI: `rrp verify <market_id>` (requires BASE_RPC_URL in `.env`).

## Verifying a batch-anchored entry (3-column)

```python
import json, hashlib

batch = json.load(open("batch_anchor.json"))

# 1. Find the entry by its digest (field 2 of the log line)
entry = next(e for e in batch["entries"] if "<digest>" in e["line"])

# 2. Verify the Merkle proof
def h(data): return hashlib.sha256(data).digest()
node = h(entry["line"].encode())
for step in entry["proof"]:
    sib = bytes.fromhex(step["sibling"])
    node = h(node + sib) if step["direction"] == "right" else h(sib + node)
assert node.hex() == batch["root"], "Merkle proof invalid"

# 3. Verify the batch tx calldata on Base
from web3 import Web3
w3 = Web3(Web3.HTTPProvider("https://mainnet.base.org"))
tx = w3.eth.get_transaction(batch["tx_hash"])
expected = b"rrp-batch:" + bytes.fromhex(batch["root"])
assert bytes(tx["input"]) == expected, "batch calldata mismatch"
print("batch anchor verified at block", batch["block_number"])
```

Or use the CLI: `rrp verify <market_id>`.

## Entries before the batch anchor

The first 22 entries (all dated 2026-07-27) were committed when the scheme was
git-witnessed only. They are retroactively covered by the batch anchor in
`batch_anchor.json`. For those entries:

- The **original commit date** is the git-witnessed timestamp (GitHub commit
  history, predating the batch anchor block).
- The **chain anchor date** is the block timestamp of the batch transaction.
- The Merkle proof in `batch_anchor.json` proves each entry was included in
  the Merkle root that was anchored on-chain.

Anyone may independently verify that: (a) those git commits predate the
resolution of any of those markets, and (b) the batch anchor tx calldata
matches the Merkle root of the 3-column log lines.

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
prose.

## What this log does *not* prove

- It does not prove a score was *good* — only that it was fixed in advance.
- It does not prove the report was scored on an honest snapshot. The report
  binds the market snapshot by hash, but the snapshot itself came from the
  Polymarket Gamma API at fetch time.
- It cannot *enforce* the disclosure policy above — only make violations
  visible. An unrevealed commitment stays permanently on the record, countable
  by anyone.
