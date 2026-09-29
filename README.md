# social-graph-community-detection

A template Hermes skill for mapping WHO belongs to WHICH circle — and who holds social power — inside a WhatsApp network, powered by [Lextrove](https://lextrove.com). Models people as nodes and observed interaction as weighted edges, then runs Leiden / Louvain community detection plus betweenness centrality. Built for the "understand the social situation" question, not routine dashboards.

## What it does

When a deeper investigation is needed (a kid under peer pressure, a community conflict, "who are the leaders here?"), the skill:

1. Defines the network boundary (bounded node set — a class, a scouts group, an invite list)
2. Collects interaction edges (DM volume, shared-group co-activity, replies, reactions)
3. Builds a weighted graph from real data (DM pairs 3×log1p; shared-group co-activity log1p of min counts)
4. Runs Leiden with a resolution sweep + a Louvain cross-check
5. Ranks centrality — betweenness finds the brokers between circles
6. Ground-truths the topology against actual message content before anything is reported

It ships with an interpretation playbook (social leaders vs cross-community pressure moves, date-conflict bridges, sudden-DM anomalies, "maybe" RSVP clusters) and a compute rule: graphs are **ad hoc** — built when there is a real question, never per cycle.

## The algorithm runner

`scripts/social_graph_leiden.py` — Leiden resolution sweep (0.5–1.5, fixed seed) + Louvain cross-check + betweenness brokers (strength-as-proximity) + a stability verdict (pairs that survive every resolution AND both algorithms). Input is a plain JSON nodes/edges spec:

```json
{ "target": "Alex", "nodes": ["Alex", "Jo"], "edges": [["Alex", "Jo", 12.5]] }
```

```bash
pip install leidenalg igraph
python3 scripts/social_graph_leiden.py graph.json --target Alex
```

A hub-based **graph builder** for watch deployments (pulls the last N days and emits the spec) ships with the companion repo [kid-social-watch](https://github.com/omernesh/kid-social-watch).

## Why a template?

This repo is deliberately generic — **no secrets, no PII, no real names, phone numbers, session names, or job IDs**. Every value (network, sessions, thresholds, language) is supplied at runtime.

## Install

```bash
git clone https://github.com/<you>/social-graph-community-detection.git
ln -sfn "$PWD/social-graph-community-detection" ~/.hermes/skills/analytics/social-graph-community-detection
```

Requires a Hermes installation with the Lextrove MCP server configured.

## Files

```
SKILL.md                            # the skill definition + full methodology
README.md                           # this file
scripts/social_graph_leiden.py      # Leiden/Louvain + brokers runner (JSON spec in)
```

## License

MIT — steal it, adapt it, ship it.
