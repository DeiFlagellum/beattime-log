# Sigelith public log — checkpoint archive

An independent copy of the signed checkpoints of the Sigelith proof-of-existence log
(<https://sigelith.org/checkpoints/>). sigelith.org writes it automatically once a week;
nothing here is edited by hand.

A checkpoint is only worth something if you can take it from someone other than the log
operator. Releases in this repository are **immutable**: once a release is published, its tag
and files cannot be changed. The repository itself could still be deleted, so every
checkpoint is also anchored in Bitcoin through OpenTimestamps and archived elsewhere.

## Naming

Sigelith is the proof infrastructure first published as *BeatTime proof* at beattime.live.
sigelith.org and beattime.live are served by one system, with one log and one signing key:
every address below also works under `https://beattime.live`, and every proof issued under the
BeatTime name keeps verifying. This repository was called `beattime-log` until 27 September
2026 and was renamed before its first release; GitHub redirects the old address.

Format identifiers inside signed data keep their original names — `beattime-proof-v1`,
`beattime-entry-v1`, `beattime-checkpoint-v1` — because they are part of the signed bytes.
Full account: <https://sigelith.org/spec/#naming>.

## Releases

One release per ISO week, tagged with the week (for example `2026-W39`). A release is
published once the OpenTimestamps proofs of the week have reached Bitcoin, and no later than
one day after the weekly checkpoint.

| file | content |
|---|---|
| `NNNNNN.json` | every checkpoint issued in that week (the daily checkpoints of its days and the weekly checkpoint issued when the week closed), byte for byte as served at `https://sigelith.org/checkpoints/NNNNNN.json` |
| `NNNNNN.json.ots` | OpenTimestamps proof for that file |
| `<week>.jsonl` | every log entry of the week, one per line (absent for a week without entries) |
| `<week>.html` | the static week page `https://sigelith.org/checkpoints/<week>/` at publication time |
| `SHA256SUMS` | SHA-256 of every file above |

## Checkpoint format — `beattime-checkpoint-v1` (frozen)

A checkpoint file is one JSON object in canonical form (RFC 8785; for the subset used here:
keys sorted, no whitespace, UTF-8, integers only, no trailing newline).

| field | meaning |
|---|---|
| `format` | `"beattime-checkpoint-v1"` |
| `kind` | `"daily"` or `"weekly"` |
| `n` | checkpoint number: 1, 2, 3… without gaps |
| `utc` | time of issue, `YYYY-MM-DDTHH:MM:SS.ffffffZ` |
| `tree_size` | number of log entries covered |
| `root` | root of the global Merkle tree over the first `tree_size` entries |
| `last_seq`, `last_chain_hash` | the last covered entry and its hash-chain link |
| `btc` | `{"height", "hash"}` of a Bitcoin block at depth 3 on which two independent sources agree, or `null`. The checkpoint was not created before that block was mined. |
| `prev` | SHA-256 of the previous checkpoint file (`null` only for `n` = 1) |
| `key` | Ed25519 public key, base64 |
| `sig` | signature, base64 |

A weekly checkpoint adds `week`, `week_size`, `week_root` (the weekly Merkle root, `null` for
a week without entries), `dump` (`{"file", "sha256"}` of the weekly dump, or `null`) and
`anchors`: bank transfers recorded since the previous weekly checkpoint, each with `week`,
`title`, `bank`, `reference` and `booked`.

```
signed = ASCII("beattime-checkpoint-v1|") || JCS(checkpoint without "sig")
sig    = base64( Ed25519(key, signed) )
file   = JCS(checkpoint including "sig")        -- no trailing newline
hash   = hex( SHA-256(file) )                   -- becomes "prev" of the next checkpoint
```

## Log entries and trees

An entry (one line of `<week>.jsonl`, or one item of `/api/proof/entries`) has `seq`,
`digest`, `utc`, `prev_chain`, `chain_hash` and `week`. `seq` is strictly increasing but may
have gaps; the position in a tree is the order, not the number.

- **Hash chain:** `chain_hash = hex(SHA-256(UTF-8(prev_chain || digest || chain_utc)))`, where
  the first `prev_chain` is 64 zeros. `chain_utc` is Python's `datetime.isoformat()` of `utc`:
  `YYYY-MM-DDTHH:MM:SS.ffffff+00:00`, **but without the fraction when the microseconds are
  exactly zero** (`YYYY-MM-DDTHH:MM:SS+00:00`). This is historical and cannot change.
- **Global tree** (`root` in checkpoints): one leaf per entry, all entries in `seq` order,
  `leaf = SHA-256(0x00 || ASCII("beattime-entry-v1|" + seq + "|" + digest + "|" + utc))`
  with `utc` exactly as in the entry. `node = SHA-256(0x01 || left || right)`; the shape is the
  Merkle Tree Hash of RFC 9162 §2.1.1, so any RFC 9162 library can check it.
- **Weekly tree** (`week_root`, bank transfer titles): the week's entries in `seq` order,
  `leaf = SHA-256(0x00 || digest as 32 bytes)`, same nodes and shape.
- **Bank transfer title:** `MROOT`, the week without the dash, and the 64-hex weekly root in
  eight groups of eight, separated by single spaces (85 characters), for example
  `MROOT 2026W38 731921cf 38dd2447 4ddd4719 489c9699 99334761 ddaa4c60 c2534cab 674b5c3e`.

## Signing keys

Kept here as well as at <https://sigelith.org/spec/#keys>, so that a release can be checked
without asking the operator anything. A new key is added here on every rotation; no key is
ever removed.

| Ed25519 public key (raw, base64) | status | signs |
|---|---|---|
| `e7y9THJIUKvNKOZHmdBjJ8E0bOKyBFVxxMpAJ8w574Y=` | current | since 2026-09-21 — every checkpoint in this repository |
| `YNVYXDyg3hQGM3F+/ec+ZNmeN1JI/hZX+CxLqyJdyN0=` | retired | 2026-06-15 to 2026-09-21 only |

The retired key was configured on two machines at once — production and a development
mirror — and the mirror signed and Bitcoin-anchored its own weekly roots for 2026-W25 and
2026-W30. A signature by that key therefore does not identify a root as ours on its own.
These two roots are **not** authoritative:

| week | rejected root |
|---|---|
| 2026-W25 | `b84714db78def4c98a429e0d0e422994cbc32e9de189fe60c96367e10a448969` |
| 2026-W30 | `7225b61edee6cd82c583759cdca1ae5155b8332cc4e806e5c88bc69c28c270b0` |

The authoritative root of every week is the one you recompute from that week's entries
(the weekly dump); for weeks with a bank anchor it is also the root named in the `MROOT`
transfer title. Full account: <https://sigelith.org/spec/#incident-2026-09>.

## Verifying a release

1. `sha256sum -c SHA256SUMS`
2. `ots verify NNNNNN.json.ots` with the OpenTimestamps client: Bitcoin attests the time by
   which the file existed.
3. Check the signature as above, then compare `key` with the key history under
   [Signing keys](#signing-keys) (the same list is at <https://sigelith.org/spec/#keys>).
4. Check the chain: the SHA-256 of each file equals `prev` of the next one, across releases too.
5. Download the whole log (`https://sigelith.org/api/proof/entries?from=1&limit=1000`, then
   follow `next`, or take the weekly dumps) and recompute the hash chain, `root` for
   `tree_size`, `last_seq` and `last_chain_hash`, and, for weekly checkpoints, `week_root`
   and the SHA-256 of the dump.
6. To check two checkpoints without downloading everything, get an RFC 9162 §2.1.4
   consistency proof from
   `https://sigelith.org/api/proof/consistency?first=<older tree_size>&second=<newer tree_size>`
   and verify it against the roots in the files from here, not against the roots echoed by
   the API.

## What a checkpoint proves, and what it does not

A checkpoint proves that Sigelith was presented with each fingerprint no later than the time
it was recorded, and that the record has not changed since. It does not prove authorship, the
truth of any content, or that anything happened. Sigelith is not a qualified trust service
provider under eIDAS. The claim is narrower and checkable: verifiable without trusting anyone,
including us.
