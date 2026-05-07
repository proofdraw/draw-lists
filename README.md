# ProofDraw — sealed entry lists (public mirror)

This repository is a **public, immutable mirror of every entry list ever sealed by [ProofDraw](https://proofdraw.com)**.

It exists for one reason: so that *anyone* can independently verify any ProofDraw draw, forever, without our infrastructure being trusted, online, or even in business.

## What's in here

Every committed file under `sha256/` is the exact byte content of a sealed entry list. The filename is the hex-encoded SHA-256 of the bytes:

```
sha256/<64-hex-char-hash>.txt
```

So `sha256/a3f1c8e9b4d27e6f….txt` is the file whose SHA-256 is `a3f1c8e9b4d27e6f…`. The path *is* the integrity check. If a verifier wants the file with hash `a3f1c8…`, they fetch the file at exactly that path. If the bytes returned don't hash to the same value, something is wrong and the proof fails — same as any ProofDraw verification.

## File format (load-bearing — do not modify in place)

Every list file is:

- Plain text, **UTF-8**, **no BOM**
- **LF** line endings (no CR, no CRLF)
- One ticket ID per line (e.g. `7K2P-9XQM`)
- Trimmed: no leading/trailing whitespace on any line
- No trailing blank line beyond a single final LF
- Tickets in the **order presented to drand**: row 0 is line 1, row 1 is line 2, etc.

Hashes are computed over the **raw bytes** of the file. Any change to encoding, line endings, ordering, or trailing whitespace changes the hash. This is enforced by `.gitattributes` (`text eol=lf`) so `git checkout` on Windows does not silently convert.

## How to verify any draw using this repo

Given a public ProofDraw commitment that names:
- A list file `sha256/<HASH>.txt`
- A drand round number `R`
- (Optional) a winning ticket `T`

…anyone can independently confirm the draw by:

```bash
# 1. Fetch the list directly from this repo
curl -sL "https://raw.githubusercontent.com/proofdraw/draw-lists/main/sha256/<HASH>.txt" -o list.txt

# 2. Confirm the bytes match the committed hash
sha256sum list.txt
# Output's first column must equal <HASH>.

# 3. Fetch the drand round
curl -s "https://api.drand.sh/public/<R>" | jq -r .randomness > rand.hex

# 4. Compute winning row
N=$(wc -l < list.txt)
ROW=$(python3 -c "print(int(open('rand.hex').read().strip(), 16) % $N)")

# 5. Read the row
sed -n "$((ROW + 1))p" list.txt
# This is the verified winner. Compare to T.
```

The same logic runs in the browser-based [proofdraw/verifier](https://github.com/proofdraw/verifier).

## Why this repo is public

The whole product pitch is: *you do not have to trust ProofDraw.* That promise is empty if the only copy of the entry list lives on our server. Our server can go down, get hacked, or be ordered to delete the file. A public Git repo on GitHub:

- Gives every sealed list a **second public timestamp** (the commit time), corroborating the X-post commitment
- Is **mirrored** automatically by GitHub-archiving services and search engines
- Is **independently auditable** — a force-push or rewrite is visible to anyone watching the repo

It is one of (eventually) several mirrors. Others may include OpenTimestamps (Bitcoin-anchored), IPFS, Arweave. The verifier can pull from any of them; the hash check makes the source irrelevant.

## What this repo will *never* do

- **Force-push.** `main` is append-only. New draws come in as new commits; existing files are never modified or removed. If you ever observe a force-push on `main`, treat all proofs in this repo as suspect and contact us immediately.
- **Delete files.** A sealed list, once committed, is in this repo forever. If a draw was created in error, that's noted in the [draw record on proofdraw.com](https://proofdraw.com), not by erasing the bytes.
- **Rewrite history.** Annotated only via new commits, never via amend/rebase of past commits.

These are operational commitments; the cryptographic proof does not depend on them, but they remove a class of "what if you secretly deleted X" questions.

## Format versioning

This is **format v1**. If we ever need to change the file format (e.g., add metadata sidecar files, support binary list bodies for non-text tickets), the change will be additive: new files alongside, old files untouched. Verifiers built against v1 will continue to work for v1 lists indefinitely.

## License

Data files (`sha256/*.txt`) are released under [CC0 1.0 Universal](./LICENSE) — public domain dedication, no rights reserved. Anyone may copy, mirror, or analyze them freely.

## Source code for the verifier

See [proofdraw/verifier](https://github.com/proofdraw/verifier) — single-file vanilla-JS verifier, MIT licensed.
