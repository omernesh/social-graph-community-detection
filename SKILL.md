---
name: social-graph-community-detection
description: "Use when mapping social circles/hierarchy in WhatsApp."
category: analytics
metadata:
  tags: [whatsapp, lextrove, graph, leiden, louvain, community-detection, social-hierarchy, ad-hoc]
  related_skills: [lextrove-analytics-queries, kid-social-watch, whatsapp-integration]
---

# Social Graph Community Detection (Ad Hoc, Leiden)

Map WHO belongs to WHICH circle and WHO holds social power inside a WhatsApp network, when a deeper investigation is needed. Kids are social creatures — to understand hierarchy in their circles, model the network and run community detection.

## When to use

- Parent asks: "who are the social leaders in this circle?" / "which subgroups exist?" / "why is this kid under pressure?"
- A wellbeing flag needs context: sudden DMs from a group-only contact, date conflicts between two events, "maybe" RSVP clusters, exclusion dynamics
- Deep-dive on a specific social situation (NOT routine daily watch)

## Compute guidance (Omer's rule)

Creating a graph per community is COMPUTE-INTENSIVE — do NOT build big graphs for every WhatsApp group on every watch cycle. Graphs are **ad hoc**: build one only when there's a specific social situation to understand. Keep the node set bounded (a class, a scouts group, a birthday invite list).

## Sources / references

- Leiden algorithm: Traag et al., "From Louvain to Leiden" — https://en.wikipedia.org/wiki/Leiden_algorithm
- Python implementations: `leidenalg` (C++ core, pip), `louvain`/`python-louvain` (pure python, pip), esclear/louvain-leiden (educational, networkx-only)
- Step-by-step Leiden in python: Swapnil Agashe's Medium guide (networkx-based)
- Dynamic community detection with Leiden: "A Starting Point for Dynamic Community Detection with Leiden Algorithm" (share.google link Omer provided)

## Pipeline

### Phase 1 — Define the network boundary
Decide the investigation scope: one group's members? A kid + their top-30 contacts? A birthday invite list? Get the member/participant list + IDs:

```
mcp_lextrove_resolve_entity(query="<group name>", entity_type="group")
mcp_lextrove_list_sessions()  # discover the child's session first, always
mcp_lextrove_get_sender_analytics(session="<child_session>", period="30d", endpoint="rankings", chat_id="<group JID>")
```

Note: `get_group` member lists can be incomplete — corroborate with sender rankings / resolve_entity participant counts.

### Phase 2 — Collect interaction edges
Nodes = people. Edges = observed interaction, weighted by volume. Sources of edges:

1. **Co-participation**: two people in the same group (edge exists, low weight)
2. **Reply chains**: consecutive messages by different senders in a group thread (medium weight)
3. **Direct DMs**: message counts between pairs (high weight)
4. **Reactions**: person A reacting to person B's messages (high weight — strong tie signal)

```
mcp_lextrove_list_messages(session="<child_session>", chat_id="<group JID>", from="<30d ago>", to=today, limit=100)
# repeat with cursor pagination; collect sender_id sequences per chat
mcp_lextrove_get_sender_analytics(session="<child_session>", endpoint="reaction-top-receivers", period="30d", sender_id="<person>")
```

Lextrove MCP is SLOW (1-5 min/call) — budget time; use delegate_task or a bash script with the HTTP fallback for multi-query pulls (see lextrove-analytics-queries skill references).

### Phase 3 — Build the graph (real-data weighting)
```python
import math, networkx as nx  # or igraph
G = nx.Graph()
# add nodes with attributes: name, is_target_child, role if known
```

**Real-data edge weights (verified 2026-09-03 on a live case):**
1. **DM volume per pair** — `get_volume_analytics(endpoint="counts", chat_type="dm", chat_id="<JID>", period="30d", session=...)` → `total`. Only possible for pairs where the monitored child is one endpoint (their session). Weight: `3 * log1p(dm_total)`.
2. **Shared-group co-activity** — for each group the child is in, `get_sender_analytics(endpoint="rankings", chat_id="<group JID>", period="30d")` gives each member's message count. For any two members of the same group, add `log1p(min(count_A, count_B))` per group. This captures that two people who both post a lot in the same group are more connected than two who merely share membership.
3. Members present in only one group still get edges to everyone else in that group via co-activity (min picks the quieter one, so a lurker gets weak edges — correct).

Cap group nodes to people who matter for the question (a 25-sender group pollutes the graph with people who post 5 msgs/30d — keep top ~15 or filter count > threshold).

### Phase 4 — Run Leiden (or Louvain)
```bash
pip install leidenalg igraph  # compiled against numpy — use a fresh venv if system numpy is 2.x and packages were built for 1.x
```
```python
import igraph as ig
import leidenalg as la
G = ig.Graph()
G.add_vertices(list_of_names)
G.add_edges([(a,b) for (a,b) in weighted_edges])  # all pairs with weight>0
G.es["weight"] = [w for (a,b,w) in weighted_edges]
# Resolution sweep — teen circles are small, but always compare several resolutions:
for res in [0.5, 0.8, 1.0, 1.2, 1.5]:
    part = la.find_partition(G, la.RBConfigurationVertexPartition,
                             weights="weight", resolution_parameter=res, seed=42)
    # NOTE (leidenalg 0.12): `part.modularity` is a FLOAT ATTRIBUTE, not a method — no parentheses.
    print(res, round(part.modularity, 4), part.membership)
# Louvain cross-check (igraph built-in):
lp = G.community_multilevel(weights="weight")
```
- **Resolution parameter**: raise (>1) for finer/smaller communities, lower (<1) for coarser. For teen social circles, start at 1.0-1.2 — cliques are small.
- **Leiden > Louvain**: guarantees well-connected communities, no badly-connected artifacts (the refinement phase). Use Leiden when available. **Compare both** — a community boundary that appears in Leiden AND Louvain AND across a resolution sweep is robust; anything that flickers is noise.
- igraph pitfalls: `G.es.select(...) + G.es.select(...)` raises TypeError (EdgeSeq has no __add__) — iterate all edges and filter instead. Missing nodes with zero edges: still add them as vertices so the partition covers the whole circle.
- **Shipped runner:** `scripts/social_graph_leiden.py` in this repo executes this phase end-to-end — resolution sweep + Louvain cross-check + betweenness brokers + a stability verdict. Feed it a JSON `{nodes, edges}` spec; a hub-based builder for watch deployments ships in the kid-social-watch repo (`scripts/social_graph_build.py`).

### Phase 5 — Centrality & hierarchy interpretation
Community structure tells you the CIRCLES. Centrality tells you the POWER:

1. **Degree / weighted degree** — who talks to everyone (popularity)
2. **Betweenness centrality** — who bridges circles (the connector — often the socially powerful one: they control information flow between subgroups)
3. **Community membership of key figures** — is the suspected "leader" in the same community as the target child, or adjacent?
4. **Bridge nodes** — people who sit between the target child's community and another (these are who pressure flows THROUGH)

```python
betweenness = nx.betweenness_centrality(G, weight='weight')
# top-N by betweenness = social brokers
```

### Phase 6 — Ground truth check
Graph metrics are HYPOTHESES — validate against actual message content before reporting:
- A high-betweenness node who messages about "choosing between parties" = confirmed broker applying pressure
- Two communities that barely interact but share one bridge = the bridge is the channel for the conflict
- A node in the target's community with high degree but whose messages are all one-directional demands = dominance signal

Quote evidence. Never report "X is the leader" from topology alone.

## Interpretation playbook (kid-social context)

- **Social leader**: high betweenness + high degree + their messages get reactions from many distinct people. But check the leader's COMMUNITY first: a high-volume person in a DIFFERENT community from the target who cold-DMs the target is running a cross-community pressure move — that is structurally different from intra-community consensus.
- **Date-conflict pressure**: two birthday-party communities overlapping → bridge person (invited to both) delivers "we don't want to choose" — the child receiving that DM is being positioned to yield; check who DIDN'T get the message (asymmetry = who the group expects to fold).
- **Sudden DM from group-only contact**: edge appears where the historical graph had none → anomalous by definition, flag it. Verify via DM counts: if `get_volume_analytics(dm counts)` ≈ 0 for 30d but a DM burst exists today, it's a cold open.
- **Bridge detection**: betweenness centrality on the weighted graph finds the connector. In the live case, the target's closest friend by DM volume had ZERO betweenness (pure same-community tie) while a friend active in both the gossip group AND the rival's group carried ALL the betweenness — she is the true broker, and the person to watch/ally.
- **"Maybe" RSVP clusters**: usually not indifference — they trace to a network-level conflict (two events, overlapping invite sets). Run the graph over the invite list to find the overlap.

## Pitfalls

- Lextrove is slow; every analytics call 1-5 min. Never run this inside the daily watch cron.
- `get_group` member lists may be incomplete — build node set from rankings + resolve counts.
- Hebrew queries: use mode=fts; hybrid can fail for Hebrew text.
- Reaction data may be sparse for some contacts — don't over-read missing reaction edges.
- Two contacts can share a display name — key by JID, not name.
- This skill is PUBLIC-template safe: no real names/numbers/session names in this file.
