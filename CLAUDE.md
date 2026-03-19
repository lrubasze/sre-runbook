# Runbooks Project

Polkadot/Substrate SRE runbooks for debugging and operating nodes, parachains, and releases.

## Structure

- `node/` — node-level debugging (sync, block production, peers, resources, finalization, collators)
- `parachain/` — chain-level issues (block production, XCM, coretime, onboarding)
- `release/` — runtime upgrade checklist
- `incident-response/` — placeholder, not yet written
- `monitoring/` — placeholder, not yet written
- `README.md` — master index, symptom lookup, observability reference (Prometheus metrics, Loki queries)

## Runbook Format

Every runbook follows this structure, layered by urgency:

1. **Quick check** — first 30 seconds: is it network-wide? is the node synced? basic role/status check
2. **Triage** — numbered steps with bold headings, plain markdown (not code blocks) so links work. Branch into sub-paths where needed (e.g., validator vs collator)
3. **Deep Investigation** — metrics checklists, relay-side verification, Loki/Prometheus queries. Only needed when triage didn't resolve it
4. **Resolution** — reference material for specific fixes (e.g., keystore, resource bottleneck, PVF). Linked from triage steps, not inlined
5. **Escalation** — what to collect before escalating, who to contact

Use `[TODO: ...]` placeholders for thresholds pending team agreement.
Prefer Prometheus metrics over log scraping — metrics work regardless of log level.
Use plain markdown (numbered lists, bold headings) for decision trees so cross-links work.

## Status

- 12 runbooks written across `node/`, `parachain/`, `release/`
- `incident-response/` and `monitoring/` are empty placeholders
- TODOs remain: Grafana dashboard URLs, escalation contacts
- Discussed but not started: `check-node.sh` health-check script
- Possible future domains: bridges, expanded XCM, infrastructure

## Writing Guidelines

- Layer by urgency: operator should know what to do in the first 30 seconds without scrolling
- Use plain markdown for triage steps (numbered lists + bold headings) — code blocks break links
- Keep code blocks for actual commands (bash, promql, logql) only
- Cross-link between related runbooks (e.g., collator ops ↔ parachain not-producing)
- Cross-link triage steps to resolution sections with markdown anchors
- Prefer metrics over logs — metrics are always available regardless of log level config
- Verify log strings against polkadot-sdk source before adding them — do not invent log messages
- Include `[TODO: ...]` placeholders for values that need team agreement (thresholds, contacts)
