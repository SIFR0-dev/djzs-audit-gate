# DJZS Audit Gate

**Audit before act.** A pre-execution reasoning gate for MetaMask Agent Wallet.

Your agent can already check whether a transaction is *safe to sign*. MetaMask
simulates it, scans it with Blockaid, and routes it through Smart Transactions.
Nothing in that pipeline asks whether the position is *justified*.

This skill adds that check. Before a swap, perp, prediction-market order, or
transfer reaches the wallet, the trade's reasoning is submitted to a
deterministic audit engine. The engine returns PASS, WAIT, or FAIL. On anything
but PASS, the action halts.

```
agent intent
   -> DJZS gate        "is this position justified?"     <- this skill
   -> MetaMask pipeline "is this transaction safe?"
   -> execution
```

Two gates, two axes. Blockaid catches malicious. This catches irrational.

## Why deterministic

The engine is not a model grading a model. A language model reads the thesis
and emits boolean flags only — did this failure mode appear, yes or no — under
three-way consensus, and any field the runs disagree on collapses to unknown.
Pure code owns every point of the score, against a frozen weight table.

Same input, same flags, same score, same hash. Every run. On anyone's machine.

Verdicts are reproducible because the weights are frozen and published: every
response carries its own taxonomy and weight hashes, so a third party can
recompute the score without trusting the operator or the agent.

## The verdicts

| Verdict | Meaning | Gate |
| --- | --- | --- |
| `PASS` | no failure modes fired, position bounded | EXECUTE |
| `WAIT` | a material field could not be resolved | HALT |
| `FAIL` | flags fired past threshold | HALT |

`WAIT` is the one worth understanding. It is not a soft fail. It means the
engine declined to guess across a gap it declared — the honest answer to an
under-specified trade. The gate halts, names the unresolved field, and lets you
supply it and re-run.

An engine that guesses when it does not know is not an auditor.

## Install

Drop the `djzs-audit-gate` folder into your agent's skills directory:

```
~/.claude/skills/djzs-audit-gate/      # Claude Code
.cursor/skills/djzs-audit-gate/        # Cursor
```

Or install via your agent's skill/plugin manager if it supports one.

Requires the `metamask-agent-wallet` skill to be installed alongside it — this
gate does not execute transactions, it only decides whether one should proceed.

## Configure

```
DJZS_MCP_ENDPOINT    https://mcp.djzs.ai/mcp     (live)
DJZS_MODE            dry-run | production        (default: dry-run)
```

**Dry-run** is unpaid and returns a verdict with no certificate. Use it for
iteration and for evaluating whether the gate belongs in your workflow.

**Production** is metered over x402 in USDC on Base Mainnet and mints a
ProofOfLogic certificate anchored to Irys. Prediction-market audits are 2.00
USDC. No accounts, no API keys, no relationship required — an agent pays, gets
audited, and carries away a certificate anyone can verify.

The skill will never trigger a paid audit without explicit per-action approval.

## Coverage

| Intent | Taxonomy | Status |
| --- | --- | --- |
| `mm predict place` | DJZS-M, 4 codes | fully live |
| `mm perps open` / `modify` | DJZS-LF v1.1, 11 codes frozen | 3 codes live |
| `mm swap execute` | DJZS-LF subset | partial |
| `mm transfer` / `send-transaction` | DJZS-LF subset | partial |

Read-only intents — balances, prices, quotes without execution — are not gated.

Partial coverage is stated rather than hidden: on those verticals fewer codes
fire, so a PASS carries less discrimination, and the skill reports that.

## What it does not do

- It does not run `mm` commands. After EXECUTE, the wallet skill does.
- It does not replicate MetaMask's simulation, threat scan, or policy.
- It does not change wallet policy. Allowlists and outflow limits are
  who/where/how-much. This audits *why*. Different axes.
- It does not guarantee a profitable trade. It checks reasoning for enumerated
  failure modes. A PASS means the thesis survived those checks, nothing more.

## The doctrine

> No capital moves unaudited. The computable gets verified. The rest gets WAIT.

Verdicts run live on Base. Certificates anchor to Irys. Replay any audit and
check the hash. Don't trust the operator — that's the point.

## Links

- djzs.ai
- MIT licensed. Issues and PRs welcome.

---

`END_TRANSMISSION. //`
