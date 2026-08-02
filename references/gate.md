# The Gate Call

Two verticals, two taxonomies, one verdict vocabulary. Route by intent type,
then map the verdict to EXECUTE or HALT.

## Routing

| `mm` intent | Vertical | Taxonomy | Status |
| --- | --- | --- | --- |
| `mm predict place` | prediction markets | DJZS-M (4 codes, sum 100) | live, paid |
| `mm perps open` / `modify` | perpetuals | DJZS-LF v1.1 (11 codes, sum 200) | frozen canon, 3 codes live |
| `mm swap execute` | spot | DJZS-LF subset | partial |
| `mm transfer`, `mm wallet send-transaction` | value movement | DJZS-LF subset | partial |

Prediction-market intents are the fully-live path. Everything else runs against
the partially-activated perp taxonomy: real, but fewer codes fire, so a PASS
carries less discrimination. Say so when reporting a verdict on a partial
vertical. Never present a thin PASS as a thorough one.

## Composing the memo

Build the audit input from the user's actual words. Do not improve it.

```
action        swap | perps-open | perps-modify | predict-place | transfer
chain         base | arbitrum | polygon | ...
asset(s)      what is being bought, sold, or wagered
size          notional, in the unit the user stated
direction     long | short | yes | no    (+ leverage if any)
thesis        WHY — the user's reasoning, verbatim where possible
exit          stop, invalidation condition, or resolution criteria
```

Two rules that matter more than they look:

**Never invent an exit.** A trade with no stated stop is itself a finding. If
the user did not give one, submit the absence. The engine has a code for it;
you supplying a plausible one destroys the signal.

**Never soften the thesis.** If the reasoning is "it's going up," submit that.
Rewriting it into something defensible produces a PASS the user did not earn.

## The call

```
tool      verify_pm_trade            (prediction markets)
payment   x402, USDC on Base Mainnet
price     2.00 USDC per production audit
mode      dry-run (unpaid, no certificate) | production (paid, certificate)
```

Endpoint and tool binding are operator-configured — see `DJZS_MCP_ENDPOINT` in
[readiness.md](readiness.md). The perp path uses the corresponding LF audit
tool on the same endpoint.

Every response self-publishes its taxonomy anchors (`pm_weights_hash`,
`pm_taxonomy_hash`). Carry them into the action record; they are what make the
verdict reproducible by a third party later.

## Verdict mapping

The engine returns tri-state. The gate returns binary.

| Verdict | Meaning | Gate |
| --- | --- | --- |
| `PASS` | no flags fired, position bounded | **EXECUTE** |
| `WAIT` | a material field could not be resolved | **HALT** |
| `FAIL` | flags fired past threshold, or a critical flag | **HALT** |

`EXECUTE` requires `PASS` **and** zero CRITICAL flags. Everything else halts.

`WAIT` is not a soft fail and must not be reported as one. It means the engine
declined to guess across a declared gap — the honest answer to an
under-specified trade. The correct response is to tell the user *which* field
was unresolved so they can supply it and re-run, not to nudge them toward
executing anyway.

## Reporting a HALT

State four things and stop:

1. the verdict (`WAIT` or `FAIL`)
2. the risk score against the threshold
3. every fired code, by its canonical name
4. for `WAIT`, the unresolved fields, by name

Do not re-frame a HALT as a partial action. Do not propose a smaller size to
sneak under a threshold. Do not re-run the audit with a softened thesis to
obtain a different verdict. A gate that can be argued with is not a gate.

## Carrying the proof

A production audit anchors a ProofOfLogic certificate to Irys. Attach to the
action record:

- `verdict_hash`
- certificate URL
- taxonomy anchors from the response

Result: the executed trade carries a permanent, publicly checkable record of
the reasoning that was approved and the ruleset version it was approved
against. Anyone can replay the input and confirm the hash without trusting the
operator or the agent.

## Downstream

`EXECUTE` is necessary, not sufficient. MetaMask's own pipeline — simulation,
Blockaid threat scan, Smart Transactions — runs after this and owns a
different question: is this transaction safe to sign. Guard policy may still
pause for 2FA. Treat `AWAITING_MFA` and a returned `pollingId` as a normal
pending state, never as a gate failure.

Two gates, two axes. This one asks whether the position is justified. That one
asks whether the transaction is safe. Neither substitutes for the other.
