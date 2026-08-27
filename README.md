# BountyCourt

Bounties whose verdict settles the money. Adjudicated by a validator jury, no oracle, no human reviewer.

**[Live demo](https://makabeez.github.io/bountycourt-app/)**

| Version | Address on Bradbury | What changed |
| --- | --- | --- |
| **v4** (current) | [`0x5bC53109B0b6b0e13b56c38aA6968a9B3875f43b`](https://explorer-bradbury.genlayer.com/address/0x5bC53109B0b6b0e13b56c38aA6968a9B3875f43b) | escrow held through a separate release; refuses to rule on truncated evidence |
| v4 (release only) | [`0x21b63DE63a2adA5B48Fc0352B89Ae4237ad79f38`](https://explorer-bradbury.genlayer.com/address/0x21b63DE63a2adA5B48Fc0352B89Ae4237ad79f38) | first step of the above, kept for the diff |
| v3 | [`0x796339E4fD619e5099E03C602f51d9aa2F2b2588`](https://explorer-bradbury.genlayer.com/address/0x796339E4fD619e5099E03C602f51d9aa2F2b2588) | escrowed payout, SHA-pinned evidence, strict field validation |
| v2 | [`0x6D42B33aCd70F0B1Cec0f56A474725727F3dF50e`](https://explorer-bradbury.genlayer.com/address/0x6D42B33aCd70F0B1Cec0f56A474725727F3dF50e) | per-criterion consensus, reward was bookkeeping |
| v1 | [`0xB639F012931a5174Fa8277762bE03bfC6645126E`](https://explorer-bradbury.genlayer.com/address/0xB639F012931a5174Fa8277762bE03bfC6645126E) | single APPROVE/REJECT verdict |

## What it does

A poster opens a bounty with acceptance criteria in plain English **and escrows the reward in the same call**. A hunter claims it by submitting a URL pinned to an immutable commit. `adjudicate()` fetches that artifact and has every validator rule on every criterion independently. The escrow is then released to whoever the agreed booleans say should have it.

The money follows the verdict, and the verdict is computed in deterministic code from what validators agreed — never from the leader's prose.

```
post_bounty(id, brief, criteria, reward)   payable — the value sent is the escrow
submit(id, evidence_url)                   https, must pin a 40-char commit SHA
adjudicate(id)                             fetch, rule per criterion, set RULED
release(id)                                pay the escrow to whoever won
get_bounty(id) · list_bounties() · get_court_name()
```

## v4 — holding value, and refusing to guess

### The verdict no longer pays out

In v3 the escrow moved inside `adjudicate`, which made a ruling final the moment it was reached. v4 stops at `RULED` and holds the value; a separate `release()` settles it.

```python
@gl.public.write
def release(self, bounty_id: str) -> str:
    bounty = self._load(bounty_id)
    if bounty["status"] != "RULED":
        raise gl.vm.UserError(f"nothing to release: bounty is {bounty['status']}")
    ...
```

Anyone may call `release`. The recipient follows from the stored per-criterion booleans, so a caller has nothing to steer, and calling it out of order reverts. Separating the payout from the ruling is what makes any contest window possible at all.

| Step | Result on chain |
| --- | --- |
| `adjudicate` | `RULED`, escrow still 1 GEN — v3 would have paid and zeroed it |
| `release` before a ruling exists | reverted, `FINISHED_WITH_ERROR` |
| `release` after `RULED` | `SETTLED`, paid to hunter |

Settlement transaction `0x410d32778c331c990065a2a80867af8415c3dd48eafdd798987d2fc7e81c2d91`.

### The jury refuses to rule on a partial view

Our [controlled study](https://github.com/Makabeez/webclaims) measured what happens when the evidence does not fit the character window: validators agreed **unanimously on answers that were false**, because the content the claims referred to never entered the window.

> Consensus guarantees agreement, not truth.

v4 checks for a filled window before convening the jury:

```python
if len(evidence) >= EVIDENCE_CHARS:
    bounty["status"] = "INCONCLUSIVE"
    ...
```

A 20,365-byte artifact against a 6,000-character window now returns:

> INCONCLUSIVE: the artifact exceeded the readable window, so the jury was not asked to rule. Escrow held. Resubmit a smaller or more specific artifact.

No ruling is requested, no value moves, and the hunter is told what to change. The guard sits between the fetch and the ruling, so it also skips the expensive half of the adjudication. Transaction `0x17ed43bb20b4ad272c440d2b672ba90fae3db07dddbeeab7f2795f57edd85e5f`.

### Still open

Appeal rounds — the losing party posting a bond to convene a second jury — are designed but not shipped. Three attempts to deploy that version failed with `FINISHED_WITH_ERROR` and no readable diagnostic. Bisecting from the accepted contract established that `release` and the truncation guard are not the cause, which narrows it but does not settle it. Rather than ship something unverified, it waits.

## Escrow

`post_bounty` is `@gl.public.write.payable`. The value sent with the call **is** the reward, so a bounty cannot promise more than it holds.

`release` computes the recipient from the agreed booleans and emits the transfer:

```python
if bounty["approved"]:
    winner = Address(bounty["hunter"])
else:
    winner = Address(bounty["poster"])

gl.get_contract_at(winner).emit_transfer(value=u256(escrow), on="finalized")
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

v4 returns **only booleans** from the jury. An earlier version also asked each validator for a per-criterion `reason` string and instructed the equivalence principle to ignore it.

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

**Settlement.** Derived in code from the agreed booleans, in `release`.

## Prompt injection

The hunter controls the artifact the contract reads, and that artifact decides whether they get paid. The ruling prompt states that anything inside the evidence block — instructions, role-play, claims of authority — is data to be judged, never instructions to follow, and evidence is delimited with explicit tags.

## Running it

No build step. `index.html` loads `genlayer-js` from esm.sh as an ES module.

```bash
git clone https://github.com/Makabeez/bountycourt-app
cd bountycourt-app
python3 -m http.server 8000
```

Wallet connection needs a real origin — `file://` will not work. Wallet discovery uses EIP-6963, because several extensions fight over `window.ethereum` and clobber each other.

The demo page reads contract state and runs the flow. Posting a bounty requires sending value, so the escrow steps are driven from the browser console — see the transaction hashes above for a complete worked cycle.

## Verifying the contract

```bash
pip install genvm-linter
genvm-lint check bounty_court.py
genvm-lint schema bounty_court.py
```

## Notes for builders

None of the following is documented.

- **Do not put free text in an answer the equivalence principle compares.** See the finding above.
- **`rc.getBalance` is unreliable.** It reported a stale balance for hours after the explorer showed the transfer complete. Read the transaction's Messages tab.
- **`waitForTransactionReceipt` is unreliable.** It has timed out on transactions that had finalized and returned early reporting `NOT_VOTED`/`IDLE`. Poll `getTransaction`.
- **`FINISHED_WITH_ERROR` does not always mean state was lost** — and it is also what a correctly-reverted guard looks like.
- **No `try/except` in contract code.** GenVM rejects `except Exception` at schema generation though `genvm-lint` accepts it. On chain the symptom is a failed deploy with a six-byte CBOR error.
- **Studio can refuse to deploy contracts Bradbury accepts.** Both the payable version and an earlier one deployed fine to Bradbury after Studio reported an error.
- **Adjudication takes 30+ minutes** — two nondet blocks across every validator.
- **Rendered HTML is a poor evidence source.** The first 6000 characters of a GitHub page are markup and navigation; the file list never arrives. Point at `api.github.com/repos/{owner}/{repo}/contents?ref=<sha>` instead.

## Files

- `bounty_court.py` — the Intelligent Contract
- `index.html` — the dApp: load a court, read bounties and per-criterion rulings

## License

MIT
