# BountyCourt v5 — Milestone submission

## Notes (Portal form)

```
Three changes to the accepted contract. The verdict is now contestable, it no longer pays out on its own, and the jury refuses to rule on evidence it cannot fully see.

1. APPEALS WITH REAL STAKES. The losing party stakes a bond equal to the escrow to reopen the case; a fresh jury rules on the same pinned evidence. Overturned, the appellant takes escrow plus bond. Upheld, both go to the original winner. Verified end to end on 0x1e68256A8Fc62c513691886a6230f00F3b4DdE7D: first ruling REJECTED (1 of 2 criteria), 1 GEN bond staked (0x4396c4b4...), second jury convened (0x455165ec...), settled "upheld: 2000000000000000000 to the original winner" - the appellant lost the bond.

2. THE RULING NO LONGER PAYS OUT. v3 settled inside adjudicate, so a verdict was final the instant it was reached. v4 sets RULED and holds the value; release() settles it, or appeal() contests it first. Calling release before a ruling reverts.

3. NO RULING ON A PARTIAL VIEW. Our study (github.com/Makabeez/webclaims) measured validators agreeing UNANIMOUSLY on false answers when evidence was truncated. Consensus guarantees agreement, not truth. A 20,365-byte artifact now returns INCONCLUSIVE with the escrow held, jury never convened (0x17ed43bb...).

Constraint found along the way: GenVM will not deploy a contract with non-deterministic blocks in two different methods - three attempts failed with no readable diagnostic. Isolated with two probes. appeal() therefore reopens the case and re-enters adjudicate rather than running its own jury, which also guarantees the second jury is genuinely independent.
```

Roughly 1,500 characters. If the field caps at 1,000, cut the constraint paragraph — it's the most interesting part but the least load-bearing for the score.

**Evidence:**
- `https://github.com/Makabeez/bountycourt-app`
- `https://explorer-bradbury.genlayer.com/address/0x1e68256A8Fc62c513691886a6230f00F3b4DdE7D`

---

## README section

Replace the "Still open" paragraph with this, and add v5 to the address table.

### Appeals

A ruling is no longer the end of it. The losing party — and only the losing party — can stake a bond equal to the escrow to reopen the case:

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
| `appeal` with 1 GEN bond | `UNDER APPEAL`, case reopened |
| `adjudicate` again | second jury reaches the same conclusion |
| settlement | `upheld: 2000000000000000000 to the original winner` |

The appellant lost the bond. That is the mechanism working: contesting a correct ruling is expensive.

### Why appeal re-enters adjudicate

The obvious design puts a second jury inside `appeal`. GenVM will not deploy that contract.

Three attempts failed with `FINISHED_WITH_ERROR` and no readable diagnostic — six bytes of CBOR on chain. Two probes isolated it: a second payable method deploys fine ([`0xD50FA562…`](https://explorer-bradbury.genlayer.com/address/0xD50FA562c5CBdeBBd3C2B9254A33618991A58405)), and a full `appeal` with bond checks, standing checks, settlement arithmetic and `emit_transfer` deploys fine ([`0x1b07f737…`](https://explorer-bradbury.genlayer.com/address/0x1b07f7377CB7C3E473c3E1C69543887ee1aea18c)) — as long as it contains no non-deterministic block.

**A contract cannot have non-deterministic blocks in two different methods.** That is not in the documentation, and it forecloses the natural design for any contract needing a second opinion.

The workaround is arguably better: `appeal` stakes the bond and reopens the case, and the caller re-enters `adjudicate`. One nondet site, two rulings, and the second jury is independent by construction because consensus runs fresh.
