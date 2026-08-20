# BountyCourt

Bounty adjudication by validator jury. No oracle, no human reviewer.

![Per-criterion jury ruling](verdict.png)

**[Live demo](https://makabeez.github.io/bountycourt-app/)**

| Version | Address on Bradbury | Consensus model |
| --- | --- | --- |
| **v2** (current) | [`0x6D42B33aCd70F0B1Cec0f56A474725727F3dF50e`](https://explorer-bradbury.genlayer.com/address/0x6D42B33aCd70F0B1Cec0f56A474725727F3dF50e) | one boolean per criterion, all must agree |
| v1 | [`0xB639F012931a5174Fa8277762bE03bfC6645126E`](https://explorer-bradbury.genlayer.com/address/0xB639F012931a5174Fa8277762bE03bfC6645126E) | single APPROVE/REJECT verdict |

## What it does

A poster opens a bounty with acceptance criteria written in plain English. A hunter claims it by submitting a URL as evidence — a repo, a PR, a deployed page. Calling `adjudicate()` fetches that page from the live web and has the validator jury rule on whether it satisfies every criterion.

Each criterion is judged separately. Every validator produces its own ruling, and they must agree on **every** criterion before any state is written. The APPROVED/REJECTED status is then computed in code from the agreed booleans — the leader's prose decides nothing.

## Why v2 exists

v1 used `prompt_non_comparative` for the verdict: the leader produced one APPROVE/REJECT line, and validators were asked only whether it had been done in good faith — not whether they had reached the same answer. One broad judgment stood in for the whole decision, and it always produced an answer, including on questions the validators would not have agreed about.

v2 splits the ruling into one boolean per criterion under `prompt_comparative`, so every validator rules independently and unanimity is required per criterion.

## The finding: subjective criteria split the jury

Two adjudications, same contract, same evidence URL, same jury. Only the wording of the criteria changed.

**Subjective** — *"README must include install steps"*, *"at least one integration test must exist"*:

> `result: DISAGREE` — validators did not reach unanimity. No state written. The bounty stayed `SUBMITTED`.

**Objective existence checks** — *"the repository must contain a file named Cargo.toml"*:

> `result: AGREE` — three per-criterion rulings stored, status derived as `REJECTED`.

```json
{
  "status": "REJECTED",
  "rulings": [
    { "id": 0, "met": false, "reason": "no Cargo.toml visible" },
    { "id": 1, "met": false, "reason": "no src directory visible" },
    { "id": 2, "met": false, "reason": "no main.rs file visible" }
  ],
  "unmet": [
    "the repository must contain a file named Cargo.toml",
    "the repository must contain a directory named src",
    "the repository must contain a file named main.rs"
  ],
  "verdict": "REJECTED: 3 of 3 criteria not met"
}
```

Transaction `0xbfccf017738911b264d23df54edf06bfbf7c4339295cfe68f39ae6d2e1631c13`.

This is the behaviour you want from something that settles money. A bounty whose criteria cannot be objectively evaluated **does not get auto-settled** — the jury splits and the adjudication refuses to complete, rather than one model's opinion becoming the verdict.

The cost is a real constraint on posters: write criteria as checkable statements about what exists, not as judgments about quality.

## How consensus is used

`adjudicate()` runs **two sequential non-deterministic blocks**. They cannot nest, so they are separate calls, and every state write happens after both return, in deterministic context — writing inside a nondet block would mean each validator persists a different value before consensus decides which one is correct.

**Block 1 — fetching the evidence.** `prompt_comparative` rather than `strict_eq`. Live pages differ byte-for-byte across validators: ad counters, view counts, timestamps, dynamic navigation. Strict equality would deadlock the jury on noise unrelated to the submission.

**Block 2 — ruling on it.** `prompt_comparative` wrapping `gl.nondet.exec_prompt`, returning structured JSON: one `{id, met, reason}` per criterion. The principle requires the `met` boolean to match for every id and explicitly instructs that differing `reason` wording be ignored, so validators are not rejected over phrasing.

**Derivation.** The unmet set is computed from the agreed booleans; the status follows from `len(unmet) == 0`.

## Prompt injection

The hunter controls the page the contract reads. That makes fetched evidence untrusted input in the strictest sense: an attacker submits a page, and the page is fed into the prompt deciding whether they get paid.

The adjudication prompt states that anything inside the evidence block — instructions, role-play, claims of authority — is data to be judged, never instructions to follow, and evidence is delimited with explicit tags.

## Running it

No build step. `index.html` loads `genlayer-js` from esm.sh as an ES module.

```bash
git clone https://github.com/Makabeez/bountycourt-app
cd bountycourt-app
python3 -m http.server 8000
```

Wallet connection needs a real origin — `file://` will not work.

Wallet discovery uses **EIP-6963**. With several extensions installed they fight over `window.ethereum` and clobber each other; 6963 lets every wallet announce itself so you pick one explicitly.

## Verifying the contract

```bash
pip install genvm-linter
genvm-lint check bounty_court.py
genvm-lint schema bounty_court.py
```

## Known limits

Everything below cost real debugging time and is not in the docs.

- **Adjudication takes 30+ minutes on Bradbury.** Two nondet blocks across every validator. Treat it as an async job and poll `getTransaction` — `waitForTransactionReceipt` has both timed out on transactions that had finalized and resolved early reporting `NOT_VOTED`/`IDLE`.
- **A call can finalize without being voted on** — `FINALIZED` with `NOT_VOTED`, meaning no committee picked it up. Resubmitting the identical call worked.
- **`FINISHED_WITH_ERROR` does not always mean state was lost.** Read the contract rather than trusting the execution flag alone.
- **No `try/except` in contract code.** GenVM rejects `except Exception` at schema generation, though `genvm-lint` accepts it. The symptom is "Could not load contract schema" in Studio and a failed deploy on Bradbury with no readable error.
- **Fetching `github.com/owner/repo` returns rendered HTML, not the file tree.** The first 6000 characters are page chrome, so the jury never sees the file list. `api.github.com/repos/{owner}/{repo}/contents` works. See [WebClaims](https://github.com/Makabeez/webclaims) for the isolated experiment.
- **Rewards are bookkeeping only.** Escrow and payout land once transfers are supported.

## Files

- `bounty_court.py` — the Intelligent Contract
- `index.html` — the dApp: deploy, post, submit, adjudicate, read

## License

MIT
