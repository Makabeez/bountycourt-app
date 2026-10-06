# BountyCourt

Bounties whose verdict settles the money — and can be contested. Adjudicated by a validator jury, no oracle, no human reviewer.

**[Live demo](https://makabeez.github.io/bountycourt-app/)**

| Version | Address | What changed |
| --- | --- | --- |
| **v6** (current, Studio Next) | [`0xDc069a180522D80F28dea7DF96FCCd121707F714`](https://explorer-studio-dev.genlayer.com/address/0xDc069a180522D80F28dea7DF96FCCd121707F714) | ported to GenVM v0.6; refuses to rule when the fetch did not return the artifact |
| v5 (Bradbury) | [`0x1e68256A8Fc62c513691886a6230f00F3b4DdE7D`](https://explorer-bradbury.genlayer.com/address/0x1e68256A8Fc62c513691886a6230f00F3b4DdE7D) | appeals with bonds — the losing party can buy a second jury |
| v4 (Bradbury) | [`0x5bC53109B0b6b0e13b56c38aA6968a9B3875f43b`](https://explorer-bradbury.genlayer.com/address/0x5bC53109B0b6b0e13b56c38aA6968a9B3875f43b) | escrow held through a separate release; refuses to rule on truncated evidence |
| v3 (Bradbury) | [`0x796339E4fD619e5099E03C602f51d9aa2F2b2588`](https://explorer-bradbury.genlayer.com/address/0x796339E4fD619e5099E03C602f51d9aa2F2b2588) | escrowed payout, SHA-pinned evidence, strict field validation |
| v2 (Bradbury) | [`0x6D42B33aCd70F0B1Cec0f56A474725727F3dF50e`](https://explorer-bradbury.genlayer.com/address/0x6D42B33aCd70F0B1Cec0f56A474725727F3dF50e) | per-criterion consensus, reward was bookkeeping |
| v1 (Bradbury) | [`0xB639F012931a5174Fa8277762bE03bfC6645126E`](https://explorer-bradbury.genlayer.com/address/0xB639F012931a5174Fa8277762bE03bfC6645126E) | single APPROVE/REJECT verdict |

Bradbury is being retired, so v6 runs on Studio Next (chain 61997). v1–v5 stay listed with their transactions as the record of how the design got here; `bounty_court.py` is v5 as deployed there, `bounty_court_studio.py` is v6.

## What it does

A poster opens a bounty with acceptance criteria in plain English **and escrows the reward in the same call**. A hunter claims it by submitting a URL pinned to an immutable commit. `adjudicate()` fetches that artifact and has every validator rule on every criterion independently.

The ruling does not pay out. The escrow is held, and either `release()` settles it or the losing party stakes a bond to contest it.

```
post_bounty(id, brief, criteria, reward)   payable — the value sent is the escrow
submit(id, evidence_url)                   https, must pin a 40-char commit SHA
adjudicate(id)                             fetch, rule per criterion, set RULED
release(id)                                pay the escrow to whoever won
appeal(id)                                 payable — bond equals escrow, reopens the case
get_bounty(id) · list_bounties() · get_court_name()
```

## v6 — no ruling on the wrong bytes

### The failure

Commit-pinned evidence guarantees the artifact cannot change. It does not guarantee the fetch returns it. A rate-limited GitHub API answers a perfectly valid pinned URL with:

```json
{"message": "API rate limit exceeded for …", "documentation_url": "https://docs.github.com/…"}
```

Every validator receives the same error object, so they agree on it perfectly — and the criteria ("the page shows a file named web_claims.py") are plainly unmet by it. Through v5, that agreement became `RULED REJECTED`, and `release()` would refund the poster: **the hunter loses the escrow for work they did, because the source had a bad minute.** Nothing in consensus can catch this. The jury is right about what it was shown.

> A ruling is only as deterministic as the bytes fetched.

### The guard

v6 checks the shape of the fetched body before convening the jury, next to the existing truncation guard:

```python
def _not_an_artifact(self, body: str) -> str:
    text = body.strip()
    low = text.lower()
    if len(text) < 64:
        return "response too short to be an artifact"
    if "rate limit exceeded" in low or "too many requests" in low:
        return "source rate-limited the request"
    if text.startswith("{") and '"message"' in text and '"documentation_url"' in text:
        return "source returned an API error object"
    if low.startswith("<!doctype html") and ("404" in low[:600] or "not found" in low[:600]):
        return "source returned an error page"
    return ""
```

A hit sets `INCONCLUSIVE` with `retryable: true`, holds the escrow, and returns the reason. `adjudicate()` accepts a retryable `INCONCLUSIVE` bounty, so once the source serves the artifact again anyone can call it and the real jury rules — a transient outage costs a retry, not the escrow. Truncation stays non-retryable: the same artifact will overflow the window again.

The checks are deterministic string tests on content every validator already agreed on in the fetch block, so the guard adds no consensus round and skips the expensive one.

### Verified on Studio Next

Both bounties carry the same two criteria and 1 GEN escrow on [`0xDc069a180522D80F28dea7DF96FCCd121707F714`](https://explorer-studio-dev.genlayer.com/address/0xDc069a180522D80F28dea7DF96FCCd121707F714).

| Bounty | Evidence | Result |
| --- | --- | --- |
| `ok-1` | `api.github.com/repos/Makabeez/webclaims/contents?ref=d190a689…` | `RULED APPROVED: all 2 criteria met`, escrow held for release or appeal — [`0x8a60067e…`](https://explorer-studio-dev.genlayer.com/tx/0x8a60067e929313c325e9f5dbbdca6669f8b5825248931b5025b0d165d3390e1c) |
| `bad-1` | same SHA, nonexistent repo → GitHub's `{"message":"Not Found","documentation_url":…}` | `INCONCLUSIVE: the fetch did not return the artifact (source returned an API error object), so the jury was not asked to rule. Escrow held.` — [`0x7b97df3c…`](https://explorer-studio-dev.genlayer.com/tx/0x7b97df3ce4a151bc1214e8cd21a8506d6721580ac932219f5d33834d82f86466) |

On v5, `bad-1` would have been ruled `REJECTED` on the error object. The guard was also exercised locally against a GitHub rate-limit body, an HTML 429 page and an empty response (all refused) and against a real directory listing and a real file object (both passed).

### Porting to GenVM v0.6

Studio Next runs GenVM v0.6. The v5 source does not deploy there unchanged:

| v5 (Bradbury) | v6 (Studio Next) |
| --- | --- |
| `from genlayer import *` | `import genlayer as gl` + `from genlayer.types import *`, Depends header pinned to `py-genlayer:5jycge4q…` |
| `class BountyCourt(gl.Contract)`, `TreeMap[str, str]` | `gl.contract.Contract`, `gl.storage.TreeMap[str, str]` |
| `prompt_comparative(fn, principle=…)` | principle **positional** — the keyword form passes schema generation and reverts at runtime |
| `gl.get_contract_at(addr).emit_transfer(value=…, on="finalized")` | `@gl.evm.contract_interface class _Recipient` then `_Recipient(addr).emit_transfer(value=…)` |

`gen_getContractSchemaForCode` against the Studio Next RPC returns the real Python traceback for a contract that will not load, which makes it a free compile check before paying for a deploy.

## Appeals

A ruling is not the end of it. The losing party — and only the losing party — stakes a bond equal to the escrow to reopen the case:

```python
@gl.public.write.payable
def appeal(self, bounty_id: str) -> str:
    ...
    if bond != escrow:
        raise gl.vm.UserError("the appeal bond must equal the escrow")
    if caller != loser:
        raise gl.vm.UserError("only the losing party may appeal")
```

The bounty returns to `SUBMITTED` and `adjudicate()` is called again, convening a fresh jury on the same pinned evidence. Overturned, the appellant takes escrow plus bond. Upheld, both go to the party that already won. Appealing costs money and losing an appeal costs more.

A complete cycle on `0x1e68256A8Fc62c513691886a6230f00F3b4DdE7D`:

| Step | Result |
| --- | --- |
| `adjudicate` | `RULED REJECTED` — 1 of 2 criteria unmet, escrow held |
| `appeal` with a 1 GEN bond | `UNDER APPEAL`, case reopened |
| `adjudicate` again | a second jury reaches the same conclusion |
| settlement | `upheld: 2000000000000000000 to the original winner` |

The appellant lost the bond. That is the mechanism working — contesting a correct ruling is expensive.

### Why appeal re-enters adjudicate

The obvious design puts a second jury inside `appeal`. GenVM will not deploy that contract.

Three attempts failed with `FINISHED_WITH_ERROR` and no readable diagnostic — six bytes of CBOR on chain. Two probes isolated it. A second payable method deploys fine ([`0xD50FA562…`](https://explorer-bradbury.genlayer.com/address/0xD50FA562c5CBdeBBd3C2B9254A33618991A58405)). A full `appeal` with bond checks, standing checks, settlement arithmetic and `emit_transfer` also deploys fine ([`0x1b07f737…`](https://explorer-bradbury.genlayer.com/address/0x1b07f7377CB7C3E473c3E1C69543887ee1aea18c)) — as long as it contains no non-deterministic block.

> **A contract cannot have non-deterministic blocks in two different methods.** That is not in the documentation, and it forecloses the natural design for anything needing a second opinion.

The workaround is arguably better. `appeal` stakes the bond and reopens the case; the caller re-enters `adjudicate`. One nondet site, two rulings, and the second jury is independent by construction because consensus runs fresh.

## The ruling no longer pays out

In v3 the escrow moved inside `adjudicate`, which made a ruling final the moment it was reached. v4 stopped at `RULED` and held the value.

Anyone may call `release`. The recipient follows from the stored per-criterion booleans, so a caller has nothing to steer, and calling it out of order reverts.

| Step | Result on chain |
| --- | --- |
| `adjudicate` | `RULED`, escrow still held |
| `release` before a ruling exists | reverted, `FINISHED_WITH_ERROR` |
| `release` after `RULED` | `SETTLED`, paid to hunter |

Settlement transaction `0x410d32778c331c990065a2a80867af8415c3dd48eafdd798987d2fc7e81c2d91`.

## No ruling on a partial view

Our [controlled study](https://github.com/Makabeez/webclaims) measured what happens when evidence does not fit the character window: validators agreed **unanimously on answers that were false**, because the content the claims referred to never entered the window.

> Consensus guarantees agreement, not truth.

The contract checks for a filled window before convening the jury. A 20,365-byte artifact against a 6,000-character window returns:

> INCONCLUSIVE: the artifact exceeded the readable window, so the jury was not asked to rule. Escrow held. Resubmit a smaller or more specific artifact.

No ruling is requested, no value moves, and the hunter is told what to change. Transaction `0x17ed43bb20b4ad272c440d2b672ba90fae3db07dddbeeab7f2795f57edd85e5f`.

## Escrow

`post_bounty` is `@gl.public.write.payable`. The value sent with the call **is** the reward, so a bounty cannot promise more than it holds.

Settlement computes the recipient from the agreed booleans and emits the transfer:

```python
gl.get_contract_at(winner).emit_transfer(value=u256(total), on="finalized")   # v5, Bradbury
_Recipient(winner).emit_transfer(value=u256(total))                             # v6, Studio Next
```

Delivery happens at finalization as a separate internal message. `rc.getBalance` reported the pre-transfer balance for hours after the explorer showed the contract at zero — read the transaction's Messages tab, not the RPC balance.

## Immutable evidence

`submit` rejects any URL that does not pin a 40-character commit SHA. A branch reference such as `?ref=main` can change between the moment a hunter submits and the moment the jury reads it, which would let the party being judged rewrite the evidence after the fact.

| Evidence URL | Result |
| --- | --- |
| `…/contents?ref=main` | rejected, nothing stored |
| `…/contents?ref=PASTE_SHA_HERE` | rejected, nothing stored |
| `…/contents?ref=d190a689…` | accepted and stored |

A second `submit` on an already-submitted bounty also reverts.

## Strict validation before value moves

Twelve checks run on the jury's output before anything is stored: the output is an object, `rulings` is a list, its length matches the criteria count, each entry is an object carrying an integer `id` within range and a genuine boolean `met`, no id appears twice, and every criterion has exactly one ruling. Any deviation reverts and the escrow stays untouched.

## The consensus finding

The jury returns **only booleans**. An earlier version also asked each validator for a per-criterion `reason` string and instructed the equivalence principle to ignore it.

That version did not reach consensus. Adjudication `0xa88a3f3f9ae66d1d9df6e2b320635cf662c876274308544e1a9afe3f2c742070` went through **four leader rotations** and ended `UNDETERMINED / DISAGREE` — even though every leader produced identical booleans. The validators were not disagreeing about the facts; they were comparing whole answers whose `reason` wording differed on every generation.

Removing the field fixed it on the first attempt, with identical claims, identical pinned evidence, and the same jury.

| Ruling format | Result |
| --- | --- |
| `{id, met, reason}` | `UNDETERMINED / DISAGREE` after 4 rotations |
| `{id, met}` | `FINALIZED / AGREE`, no rotations |

> Free text in a compared answer prevents consensus even when the substantive verdict is identical. Do not emit what the equivalence principle then has to discount.

## How consensus is used

`adjudicate()` runs **two sequential non-deterministic blocks**. They cannot nest, so they are separate calls, and every state write happens after both return, in deterministic context — writing inside a nondet block means each validator persists a different value before consensus decides which is correct.

**Block 1 — read the artifact.** `prompt_comparative` with a tolerant principle. Even an immutable artifact can differ in whitespace between fetches; `strict_eq` would deadlock the jury on noise.

**Block 2 — rule on each criterion.** `prompt_comparative` wrapping `gl.nondet.exec_prompt`, returning `{"rulings": [{"id": 0, "met": true}]}`. The principle: same ids, same booleans, nothing else matters.

**Settlement.** Derived in code from the agreed booleans — in `release`, or in the appeal branch of `adjudicate`.

## Prompt injection

The hunter controls the artifact the contract reads, and that artifact decides whether they get paid. The ruling prompt states that anything inside the evidence block — instructions, role-play, claims of authority — is data to be judged, never instructions to follow, and evidence is delimited with explicit tags.

## Running it

No build step. `index.html` loads `genlayer-js@2.0.0-rc.1` from esm.sh, falling back to jsDelivr, and reads v6 on Studio Next.

```bash
git clone https://github.com/Makabeez/bountycourt-app
cd bountycourt-app
python3 -m http.server 8000
```

Wallet connection needs a real origin — `file://` will not work. Wallet discovery uses EIP-6963, because several extensions fight over `window.ethereum` and clobber each other.

The demo page reads contract state and renders per-criterion rulings. Posting a bounty and appealing require sending value, so those steps are driven from the browser console — the transaction hashes above are complete worked cycles.

## Verifying the contract

```bash
pip install genvm-linter
genvm-lint check bounty_court.py
genvm-lint schema bounty_court.py
```

`genvm-lint` targets the v5 API. For v6, call `gen_getContractSchemaForCode` on `https://studio-next.genlayer.com/api` with the file contents: it returns the schema, or the traceback GenVM would hit.

Note that `genvm-lint` passes several constructs GenVM rejects at deploy. See below.

## Notes for builders

None of the following is documented.

- **No non-deterministic blocks in two different methods.** Deployment fails with no readable diagnostic. Re-enter a single adjudication path instead.
- **Do not put free text in an answer the equivalence principle compares.** It prevents consensus even when the substantive verdict is identical.
- **No `try/except` in contract code.** GenVM rejects `except Exception` at schema generation though `genvm-lint` accepts it.
- **`rc.getBalance` is unreliable.** It reported a stale balance for hours after the explorer showed the transfer complete.
- **`waitForTransactionReceipt` is unreliable.** It has timed out on transactions that had finalized and returned early reporting `NOT_VOTED`/`IDLE`. Poll `getTransaction`.
- **`FINISHED_WITH_ERROR` does not always mean state was lost** — and it is also what a correctly-reverted guard looks like.
- **Studio can refuse to deploy contracts Bradbury accepts.**
- **Adjudication takes 30+ minutes** — two nondet blocks across every validator.
- **A pinned URL does not pin the response.** Rate limits and API errors come back as well-formed bodies every validator agrees on. Check the shape of what was fetched before asking the jury about it.
- **On GenVM v0.6, pass the equivalence principle positionally.** `principle=` is accepted at schema time and fails at runtime.
- **Rendered HTML is a poor evidence source.** The first 6000 characters of a GitHub page are markup and navigation. Point at `api.github.com/repos/{owner}/{repo}/contents?ref=<sha>` instead.

## Files

- `bounty_court_studio.py` — v6, the Intelligent Contract live on Studio Next
- `bounty_court.py` — v5 as deployed on Bradbury
- `index.html` — landing page and viewer: load a court, read bounties and per-criterion rulings

## License

MIT
