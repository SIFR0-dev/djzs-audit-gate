---
name: djzs-audit-gate
description: Use BEFORE executing any capital-deploying MetaMask Agent Wallet action — a swap, bridge, perpetual open/modify, prediction-market order, or a transfer that moves value. Audits the trade thesis with the DJZS adversarial logic auditor and returns EXECUTE or HALT before the `mm` command runs. This is the upstream gate; MetaMask's own simulation/Blockaid/Smart-Transactions pipeline is the downstream one. Audit-before-act.
license: MIT
metadata:
  author: SIFR0-dev
  version: "1.0.0"
  product: DJZS Protocol
  pairsWith: metamask-agent-wallet
  targetsAgentWallet: "2.0.0"
---

# DJZS Audit Gate

DJZS answers **"should I take this position at all?"** It does **not** answer "is
this transaction safe to sign?" — MetaMask's mandatory pipeline (simulation →
Blockaid threat scan → Smart Transactions) owns that, downstream. Two
independent gates on two different axes. This skill is the first one.

**Invariant:** no capital-deploying action reaches the `metamask-agent-wallet`
skill until DJZS returns `EXECUTE`. On `HALT`, stop and report — do not execute.

## When this fires

Any time the user's request would result in the `metamask-agent-wallet` skill
running one of: `mm swap execute`, `mm perps open` / `modify`, `mm predict place`,
`mm transfer`, `mm wallet send-transaction`. Read-only intents (balance, price,
history, quotes without execution) do **not** require the gate.

## Decision flow

1. **Readiness** (first gated action of a session only). Confirm the environment
   is wired. See [references/readiness.md](references/readiness.md). If anything
   is missing, surface it and stop — do not improvise around a broken setup.
2. **Capture the intent.** Build the trade as: action, chain, asset(s), size,
   direction/leverage if any, the **thesis** (why), and the **exit plan / stop**.
   A trade with no articulated thesis or no stated exit is itself signal — pass
   it through as stated; never invent an exit the user did not give.
3. **Audit.** Send the composed memo to the DJZS oracle. See
   [references/gate.md](references/gate.md) for the exact call, tiers, and
   payment. Two paths:
   - **Dry-run (unpaid signal):** for iteration. No certificate, no spend.
   - **Production (paid, x402 USDC):** mints a ProofOfLogic certificate. **Costs
     real money — see the safety rule below.**
4. **Gate on the verdict.**
   - `EXECUTE` ⟺ verdict is `PASS` **and** zero `CRITICAL` flags. Hand off to
     `metamask-agent-wallet` with the original intent.
   - Otherwise `HALT`. Report the verdict, risk score, and fired flag codes.
     Do not execute. Do not soften or re-frame a HALT into a partial action.
5. **Carry the proof.** On a production audit, attach the returned Irys
   certificate URL and logic hash to the action record, so the executed trade is
   backed by a permanent, public audit.

A returned `EXECUTE` is necessary but **not sufficient** for the trade to land:
MetaMask's Guard policy may still pause it for 2FA (out-of-allowlist, outflow
limit). Treat MetaMask's `AWAITING_MFA` / `pollingId` as a normal pending state,
not a DJZS failure.

## Safety rule (non-negotiable)

A **production** audit is a paid call: it spends real USDC over x402 on Base
Mainnet. **Never trigger a paid audit without explicit, per-action user
approval.** State the tier and its price, get a clear yes, then call. Default to
the dry-run path for anything exploratory. If payment is not configured, say so
and offer the dry-run — do not silently fall back or retry against a paid route.

## Routing

| Intent | Action | Reference |
| --- | --- | --- |
| Verify the gate environment is ready | `mm doctor` + oracle health | [readiness.md](references/readiness.md) |
| Audit a prediction-market order (`mm predict place`) | DJZS-M path, fully live | [gate.md](references/gate.md) |
| Audit a perp / swap / transfer | DJZS-LF path, partially live | [gate.md](references/gate.md) |
| Hand off an approved trade | defer to `metamask-agent-wallet` skill | (that skill's routing) |

Prediction markets run against the complete live taxonomy. The other verticals
run against a partially-activated one — fewer codes fire, so a PASS carries
less discrimination. Report that distinction; never present a thin PASS as a
thorough one.

## What this skill does not do

- It does not run `mm` commands. After `EXECUTE`, the `metamask-agent-wallet`
  skill constructs and runs the command.
- It does not replicate MetaMask's threat scan, simulation, or policy. Those run
  downstream regardless.
- It does not change MetaMask wallet policy. Policy (allowlists, outflow limits)
  is who/where/how-much; DJZS audits why. Different axes.
