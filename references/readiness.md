# Readiness

Run once per session, before the first gated action. If anything here fails,
surface it and stop. Do not improvise around a broken setup and do not fall
back to an ungated execution path.

## 1. Wallet side

```
mm doctor
```

Confirms the MetaMask Agent Wallet CLI is installed, a session exists, and the
wallet is reachable. A failure here is a wallet problem, not a gate problem —
report it and let the user fix it before continuing.

## 2. Oracle side

Confirm the DJZS endpoint responds and reports its taxonomy anchors. Every
response from the audit tool self-publishes its weight and taxonomy hashes; if
those are absent, the endpoint is not the production engine and the gate must
not be treated as authoritative.

```
DJZS_MCP_ENDPOINT   <-- set by the operator; required
DJZS_MODE           dry-run | production   (default: dry-run)
```

Health signal to check for:

- endpoint reachable
- response carries `pm_weights_hash` and `pm_taxonomy_hash`
- engine version / commit reported

## 3. Payment side (production mode only)

A production audit spends real USDC over x402 on Base Mainnet. Before the
first production call of a session, confirm:

- an x402-capable wallet or facilitator is configured
- the operator has explicitly approved paid audits for this session
- the per-audit price is known and has been stated to the user

If payment is not configured, say so plainly and offer the dry-run path. Never
silently retry a failed paid call, and never fall back from dry-run to paid.

## 4. Scope check

The gate applies to capital-deploying actions only. Read-only intents —
balances, prices, history, quotes without execution — skip the gate entirely.
Do not gate what does not move value; it wastes the user's time and, in
production mode, their money.
