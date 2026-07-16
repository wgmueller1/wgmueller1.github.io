---
layout: post
title: "Beyond Tokenomics: A Framework for Monitoring AI Costs at Enterprise Scale"
description: "Why $/token dashboards keep surprising CEOs, and what a small experiment clustering my own Claude prompt history suggests about monitoring AI spend properly"
modified: 2026-07-11
tags: [ai,cost,monitoring,finops,embeddings,clustering,anomaly,detection,enterprise]
comments: true
published: true
image:
  feature:
  credit:
  creditlink:
---

Uber's CTO recently disclosed that the company burned through its <em>entire 2026 AI coding budget in four months</em>. By March, 84% of Uber's engineers had adopted Claude Code, and roughly 70% of newly committed code at the company was AI-authored — impressive adoption, and an entirely predictable way to blow through a budget that was sized for a different usage pattern. One unnamed enterprise reportedly ran up a <strong>$500 million bill in a single month</strong> on Claude usage after nobody had set per-employee usage caps. Nvidia's Bryan Catanzaro has said plainly that for many of his own workflows, "the cost of compute...now far exceeds what the company spends on the employees using it." Jensen Huang's own recommendation — a $250,000 annual token budget per $500,000 engineer — is itself an admission that AI inference is now a line item comparable to headcount, not a rounding error under it. (<a href="https://www.forbes.com/sites/jemmagreen/2026/07/02/ai-costs-more-than-the-people-it-replaced/" target="_blank">Forbes</a>)

None of this is a fringe problem. <a href="https://www.techradar.com/pro/ceos-are-being-left-baffled-at-the-high-cost-of-moving-to-ai-shockingly-enough-sacking-human-workers-isnt-resulting-in-huge-savings" target="_blank">TechRadar reported</a> on a KPMG survey in which 29% of senior leaders across 20 countries said they were struggling to understand the rise in their own operating costs as they scaled AI, and close to half of organizations surveyed said they'd had to rephase AI deployments after costs outweighed the expected value. An MIT study cited in the same reporting found AI automation is currently economically justified in only about 23% of roles it's being pointed at. Companies are laying off workers to fund tools that, in a lot of these cases, cost more than the workers they replaced.

I don't think this is because AI is inherently unaffordable. I think it's because most organizations are monitoring AI spend the same crude way you'd monitor a phone bill — total tokens times a per-million rate — and that kind of tokenomics genuinely cannot answer the questions a CEO ends up asking after the invoice arrives: which team, which workflow, and was this normal. Below is a small, personal-scale experiment in the kind of monitoring that actually would answer those questions, plus what I think the enterprise version of it needs to look like.

## Why Tokenomics Alone Fails

A dashboard that reports "$40k this month" tells you almost nothing useful. A few reasons a raw dollars-and-tokens total keeps blindsiding the people responsible for it:

- **Model mix.** 95% of enterprise AI usage reportedly still runs on the costliest frontier models, often by default rather than by decision. Total spend doesn't tell you whether that's appropriate — it takes knowing what each task actually needed to know whether a cheaper model would've done the job.
- **Caching.** Prompt caching can cut effective cost by an order of magnitude on repeated context, but a cache-read token, a cache-write token, and a fresh input token are all priced differently. Two teams can show identical raw token counts and have wildly different bills depending on cache hit rate.
- **Agentic fan-out.** A single request today can spawn dozens of tool calls, sub-agent invocations, and retries — which is exactly the shape of Uber's problem: high adoption (84% of engineers) times high automation (70% of code) times no visibility into what any of that fan-out was costing per unit of shipped value. Attributing cost to "the prompt" undercounts badly when the prompt is really the seed of a much larger task.
- **Attribution.** None of the above matters if a request can't be tied back to a team, project, or use case. The $500 million-in-a-month story isn't really a story about AI being expensive — it's a story about nobody having set a cap, which is only possible when nobody's tracking usage by who's generating it.

Fixing this requires seeing spend the same way you'd see any other high-volume, unlabeled event stream: you need to attribute it, cluster it by what it's actually for, and watch it for the kind of statistical anomaly that a budget alert set at a fixed dollar threshold will always catch too late. That's a data problem I already had a smaller version of sitting on my own laptop.

## A Small-Scale Proof of Concept

I don't have access to an enterprise's AI usage logs, but I have unrestricted access to my own — every prompt I've sent Claude, across both Claude Code and the Claude.ai web app, going back to mid-2023. So I built the smallest version of a use-case-level cost monitor I could: instead of asking "how much did I spend," I asked "what was I actually spending it on," and let the data answer that itself rather than tagging things by hand.

### Collecting the Data

Claude Code keeps a local history: a top-level `history.jsonl` plus a per-project conversation transcript for every session, with exact input/output token counts since the CLI logs what it actually sends to the API. I paired that with an export of my Claude.ai web conversations, which don't carry the same token accounting, so I estimated token counts there with a tokenizer and tagged every row with its source (`exact` vs. `estimated`) so the two wouldn't get silently conflated later — the same distinction an enterprise system would need between metered API usage and everything else.

Both sources got flattened into a single CSV — prompt text, project, timestamp, character/token counts, and source — leaving 463 prompts after dropping empty rows.

### Embedding and Clustering

Rather than pick a number of use-case categories up front — which is what most cost dashboards do when they let you tag spend by hand-picked labels — I let the structure emerge from the prompts themselves, the same underlying idea as my earlier <a href="/community-detection-viz/">SBM/GNN community detection</a> post:

1. Each prompt gets embedded with `sentence-transformers` (`all-MiniLM-L6-v2`).
2. I build an implicit similarity graph by thresholding cosine similarity between embeddings (0.55) and run community detection over it — prompts that are semantically close enough end up in the same community; everything else is left unclustered rather than force-fit into a group.
3. For visualization only, I reduce the same embeddings to 2D with UMAP. The clustering itself never sees the 2D coordinates — they're purely a layout for plotting, not part of the community detection.

That produced 80 communities out of 463 prompts, with about a quarter sitting as singletons — one-off questions with no close semantic neighbors. At enterprise scale, this is the step that replaces manual cost-center tagging: instead of a human deciding upfront that spend gets bucketed by team or by project code, the clustering discovers the actual shape of the usage, then you can attribute cost to that shape.

## The Result

The chart below is built from the real clustering output — real positions, real cluster sizes, real token counts — but I've stripped out the individual prompt text and project names before publishing it. Hovering a point shows you which topic group it belongs to and its token cost, not what I actually typed.

<div id="cluster-viz"></div>

<style>
#cluster-viz {
  --surface-1: #fcfcfb;
  --page: #f9f9f7;
  --ink-1: #14140f;
  --ink-2: #52514e;
  --ink-m: #898781;
  --grid: #e1e0d9;
  --border: rgba(11,11,11,0.10);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  font-size: 14px;
  color: var(--ink-1);
  background: var(--page);
  border-radius: 8px;
  padding: 20px;
  margin: 24px 0;
}
#cluster-viz * { box-sizing: border-box; }
#cluster-viz .stat-row { display: flex; gap: 24px; flex-wrap: wrap; margin-bottom: 14px; }
#cluster-viz .stat { display: flex; flex-direction: column; gap: 2px; }
#cluster-viz .stat-val { font-size: 22px; font-weight: 600; color: var(--ink-1); font-variant-numeric: tabular-nums; }
#cluster-viz .stat-lbl { font-size: 11px; color: var(--ink-m); text-transform: uppercase; letter-spacing: 0.05em; }
#cluster-viz .chart-wrap { background: var(--surface-1); border: 1px solid var(--border); border-radius: 8px; overflow: hidden; position: relative; margin-bottom: 12px; }
#cluster-viz #scatter { display: block; width: 100%; height: 460px; cursor: crosshair; }
#cluster-viz #tooltip {
  position: fixed; pointer-events: none; background: var(--surface-1); border: 1px solid var(--border);
  border-radius: 6px; padding: 10px 12px; max-width: 280px; font-size: 12px; line-height: 1.5;
  box-shadow: 0 4px 16px rgba(0,0,0,0.15); display: none; z-index: 100;
}
#cluster-viz #tooltip .tt-cat { color: var(--ink-1); font-weight: 600; margin-bottom: 4px; }
#cluster-viz #tooltip .tt-meta { color: var(--ink-2); }
#cluster-viz .legend { display: flex; flex-wrap: wrap; gap: 6px 14px; padding: 0 4px 4px; }
#cluster-viz .legend-item { display: flex; align-items: center; gap: 6px; font-size: 12px; color: var(--ink-2); cursor: pointer; padding: 2px 4px; border-radius: 3px; }
#cluster-viz .legend-item:hover { background: var(--grid); }
#cluster-viz .legend-item.dimmed { opacity: 0.35; }
#cluster-viz .legend-dot { width: 10px; height: 10px; border-radius: 50%; flex-shrink: 0; }
#cluster-viz .cost-wrap { background: var(--surface-1); border: 1px solid var(--border); border-radius: 8px; overflow-x: auto; margin-top: 14px; }
#cluster-viz .cost-wrap h2 { font-size: 12px; font-weight: 600; color: var(--ink-2); text-transform: uppercase; letter-spacing: 0.06em; padding: 12px 14px 8px; }
#cluster-viz .cost-table { width: 100%; border-collapse: collapse; font-size: 13px; font-variant-numeric: tabular-nums; }
#cluster-viz .cost-table th { padding: 6px 12px; text-align: left; font-size: 11px; color: var(--ink-m); text-transform: uppercase; letter-spacing: 0.05em; border-bottom: 1px solid var(--grid); cursor: pointer; user-select: none; white-space: nowrap; }
#cluster-viz .cost-table th:hover { color: var(--ink-1); }
#cluster-viz .cost-table th.sort-asc::after { content: ' \2191'; }
#cluster-viz .cost-table th.sort-desc::after { content: ' \2193'; }
#cluster-viz .cost-table td { padding: 6px 12px; border-bottom: 1px solid var(--grid); color: var(--ink-1); vertical-align: middle; white-space: nowrap; }
#cluster-viz .cost-table tr:last-child td { border-bottom: none; }
#cluster-viz .cost-table .num { text-align: right; }
#cluster-viz .cost-table .cost-val { text-align: right; font-weight: 500; }
#cluster-viz .swatch { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 7px; vertical-align: middle; }
#cluster-viz .cost-note { font-size: 11px; color: var(--ink-m); padding: 8px 14px 10px; }
</style>

<script type="application/json" id="cluster-viz-data">{"points":[{"group":-1,"x":6.225,"y":5.733,"output_tokens":0,"input_tokens":7,"token_source":"estimated","year":"2023"},{"group":-1,"x":4.825,"y":5.992,"output_tokens":88,"input_tokens":17,"token_source":"estimated","year":"2025"},{"group":-1,"x":4.693,"y":6.682,"output_tokens":168,"input_tokens":15,"token_source":"estimated","year":"2025"},{"group":-1,"x":4.658,"y":7.054,"output_tokens":112,"input_tokens":7,"token_source":"estimated","year":"2025"},{"group":-1,"x":1.205,"y":8.767,"output_tokens":155,"input_tokens":3,"token_source":"estimated","year":"2025"},{"group":-2,"x":5.603,"y":3.209,"output_tokens":241,"input_tokens":45,"token_source":"estimated","year":"2025"},{"group":-1,"x":5.634,"y":3.155,"output_tokens":269,"input_tokens":18,"token_source":"estimated","year":"2025"},{"group":-1,"x":5.801,"y":3.634,"output_tokens":198,"input_tokens":13,"token_source":"estimated","year":"2025"},{"group":-2,"x":5.762,"y":3.162,"output_tokens":322,"input_tokens":94,"token_source":"estimated","year":"2025"},{"group":-2,"x":5.665,"y":3.07,"output_tokens":223,"input_tokens":655,"token_source":"estimated","year":"2025"},{"group":-2,"x":5.93,"y":3.262,"output_tokens":278,"input_tokens":8,"token_source":"estimated","year":"2025"},{"group":-2,"x":5.662,"y":3.154,"output_tokens":259,"input_tokens":89,"token_source":"estimated","year":"2025"},{"group":-1,"x":5.419,"y":3.429,"output_tokens":353,"input_tokens":6,"token_source":"estimated","year":"2025"},{"group":-2,"x":1.25,"y":8.792,"output_tokens":345,"input_tokens":1,"token_source":"estimated","year":"2025"},{"group":-1,"x":5.784,"y":3.029,"output_tokens":127,"input_tokens":23,"token_source":"estimated","year":"2025"},{"group":2,"x":6.915,"y":2.387,"output_tokens":276,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.018,"y":2.318,"output_tokens":281,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":2,"x":7.02,"y":2.316,"output_tokens":153,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":2,"x":7.101,"y":2.245,"output_tokens":5501,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":2,"x":7.197,"y":1.891,"output_tokens":108,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":2,"x":7.249,"y":1.93,"output_tokens":95,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":2,"x":6.759,"y":1.935,"output_tokens":192,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":2,"x":6.702,"y":1.974,"output_tokens":5579,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":2,"x":7.267,"y":1.897,"output_tokens":373,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.223,"y":2.43,"output_tokens":112,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.951,"y":2.454,"output_tokens":104,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":8,"x":2.837,"y":2.401,"output_tokens":98,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.321,"y":2.235,"output_tokens":102,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.656,"y":2.32,"output_tokens":126,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.932,"y":4.236,"output_tokens":367,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.836,"y":5.411,"output_tokens":301,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":0,"x":6.484,"y":9.467,"output_tokens":123,"input_tokens":80,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.75,"y":2.31,"output_tokens":199,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":8,"x":2.95,"y":2.096,"output_tokens":113,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":2.979,"y":2.211,"output_tokens":267,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":8,"x":3.029,"y":2.565,"output_tokens":89,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":5,"x":4.139,"y":3.668,"output_tokens":159,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":5,"x":4.26,"y":3.617,"output_tokens":9379,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":4.061,"y":5.608,"output_tokens":2183,"input_tokens":847,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.457,"y":7.274,"output_tokens":327,"input_tokens":357,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.197,"y":8.808,"output_tokens":302,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.299,"y":6.983,"output_tokens":351,"input_tokens":42,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.535,"y":6.935,"output_tokens":302,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.116,"y":7.684,"output_tokens":294,"input_tokens":29,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.773,"y":7.073,"output_tokens":290,"input_tokens":24,"token_source":"estimated","year":"2026"},{"group":1,"x":5.921,"y":7.71,"output_tokens":299,"input_tokens":69,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.94,"y":7.102,"output_tokens":507,"input_tokens":26,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.01,"y":7.17,"output_tokens":504,"input_tokens":26,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.104,"y":5.948,"output_tokens":602,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.867,"y":5.99,"output_tokens":463,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.564,"y":4.036,"output_tokens":365,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.042,"y":5.495,"output_tokens":361,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.633,"y":2.025,"output_tokens":16677,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.364,"y":2.194,"output_tokens":247,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.459,"y":2.119,"output_tokens":253,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.756,"y":1.848,"output_tokens":283,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.905,"y":1.831,"output_tokens":272,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.706,"y":1.778,"output_tokens":249,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.928,"y":1.841,"output_tokens":209,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.258,"y":1.725,"output_tokens":153,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.817,"y":1.613,"output_tokens":158,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.8,"y":1.665,"output_tokens":6862,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.474,"y":2.295,"output_tokens":212,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":3,"x":9.365,"y":2.06,"output_tokens":16702,"input_tokens":37,"token_source":"estimated","year":"2026"},{"group":3,"x":9.007,"y":1.919,"output_tokens":105,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.686,"y":3.088,"output_tokens":55,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":9,"x":6.307,"y":2.945,"output_tokens":77,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.09,"y":2.913,"output_tokens":481,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-1,"x":8.19,"y":2.419,"output_tokens":23957,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.49,"y":2.685,"output_tokens":5781,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.648,"y":3.326,"output_tokens":13146,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":8,"x":2.993,"y":2.418,"output_tokens":156,"input_tokens":21,"token_source":"estimated","year":"2026"},{"group":3,"x":9.327,"y":2.112,"output_tokens":645,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":3,"x":9.237,"y":2.148,"output_tokens":31392,"input_tokens":26,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.209,"y":8.747,"output_tokens":7123,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":3,"x":9.53,"y":2.581,"output_tokens":1300,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":-1,"x":8.011,"y":3.667,"output_tokens":33411,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.285,"y":3.56,"output_tokens":22263,"input_tokens":23,"token_source":"estimated","year":"2026"},{"group":2,"x":7.099,"y":2.095,"output_tokens":37,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.05,"y":9.322,"output_tokens":82,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.781,"y":9.422,"output_tokens":48,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.239,"y":8.811,"output_tokens":460,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.968,"y":2.308,"output_tokens":346,"input_tokens":2,"token_source":"estimated","year":"2026"},{"group":-1,"x":1.416,"y":8.454,"output_tokens":1393,"input_tokens":3,"token_source":"estimated","year":"2026"},{"group":2,"x":7.386,"y":1.968,"output_tokens":28,"input_tokens":35,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.629,"y":8.826,"output_tokens":594,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":2,"x":7.134,"y":2.154,"output_tokens":20,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":2,"x":6.899,"y":2.065,"output_tokens":49,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.89,"y":4.593,"output_tokens":108,"input_tokens":55,"token_source":"estimated","year":"2026"},{"group":9,"x":6.845,"y":3.253,"output_tokens":58,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":4,"x":8.143,"y":3.128,"output_tokens":571,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.663,"y":3.486,"output_tokens":21360,"input_tokens":57,"token_source":"estimated","year":"2026"},{"group":2,"x":7.439,"y":2.572,"output_tokens":16959,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":2,"x":7.367,"y":2.566,"output_tokens":535,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.595,"y":3.686,"output_tokens":16434,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":9.271,"y":2.974,"output_tokens":399,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.024,"y":3.873,"output_tokens":134,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.658,"y":9.742,"output_tokens":130,"input_tokens":2,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.952,"y":3.875,"output_tokens":690,"input_tokens":47,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.82,"y":3.91,"output_tokens":1395,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.366,"y":3.52,"output_tokens":202,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.015,"y":5.196,"output_tokens":4779,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.042,"y":5.484,"output_tokens":13,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.778,"y":5.91,"output_tokens":0,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.03,"y":3.854,"output_tokens":612,"input_tokens":40,"token_source":"estimated","year":"2026"},{"group":0,"x":6.258,"y":9.186,"output_tokens":217,"input_tokens":183,"token_source":"estimated","year":"2026"},{"group":6,"x":7.401,"y":5.764,"output_tokens":22823,"input_tokens":43,"token_source":"estimated","year":"2026"},{"group":1,"x":5.263,"y":7.463,"output_tokens":465,"input_tokens":29,"token_source":"estimated","year":"2026"},{"group":1,"x":5.339,"y":6.351,"output_tokens":4775,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":1,"x":4.824,"y":6.977,"output_tokens":8120,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":1,"x":5.101,"y":7.128,"output_tokens":402,"input_tokens":39,"token_source":"estimated","year":"2026"},{"group":1,"x":5.011,"y":7.334,"output_tokens":4683,"input_tokens":22,"token_source":"estimated","year":"2026"},{"group":1,"x":4.697,"y":6.428,"output_tokens":4732,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":1,"x":5.106,"y":7.498,"output_tokens":5959,"input_tokens":21,"token_source":"estimated","year":"2026"},{"group":1,"x":5.01,"y":7.539,"output_tokens":5939,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":1,"x":5.165,"y":7.059,"output_tokens":6557,"input_tokens":29,"token_source":"estimated","year":"2026"},{"group":1,"x":6.111,"y":3.686,"output_tokens":87,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":1,"x":5.99,"y":3.444,"output_tokens":8905,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":1,"x":6.16,"y":2.785,"output_tokens":105,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.08,"y":5.852,"output_tokens":14982,"input_tokens":46,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.114,"y":5.545,"output_tokens":5957,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.788,"y":3.099,"output_tokens":1625,"input_tokens":48,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.452,"y":5.364,"output_tokens":71,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.899,"y":5.416,"output_tokens":324,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.773,"y":3.067,"output_tokens":333,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.094,"y":2.97,"output_tokens":203,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.905,"y":3.04,"output_tokens":276,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-1,"x":1.213,"y":8.757,"output_tokens":16,"input_tokens":2,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.768,"y":3.057,"output_tokens":363,"input_tokens":43,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.866,"y":3.232,"output_tokens":348,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.943,"y":2.985,"output_tokens":313,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":8,"x":3.052,"y":2.555,"output_tokens":185,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.21,"y":2.964,"output_tokens":188,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.199,"y":2.888,"output_tokens":74,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.112,"y":3.567,"output_tokens":7279,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":6,"x":7.367,"y":5.75,"output_tokens":5628,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.867,"y":3.989,"output_tokens":710,"input_tokens":39,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.076,"y":4.863,"output_tokens":19932,"input_tokens":23,"token_source":"estimated","year":"2026"},{"group":6,"x":7.184,"y":5.737,"output_tokens":294,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":6,"x":7.155,"y":5.755,"output_tokens":440,"input_tokens":34,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.808,"y":5.772,"output_tokens":400,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.569,"y":3.488,"output_tokens":1456,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":1,"x":5.107,"y":7.041,"output_tokens":7035,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.76,"y":2.122,"output_tokens":11008,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.463,"y":2.531,"output_tokens":2868,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":1,"x":5.968,"y":8.95,"output_tokens":183,"input_tokens":362,"token_source":"estimated","year":"2026"},{"group":1,"x":7.122,"y":3.235,"output_tokens":205,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":1,"x":7.209,"y":3.226,"output_tokens":1276,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":1,"x":7.127,"y":3.051,"output_tokens":2154,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":1,"x":5.957,"y":2.141,"output_tokens":615,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":0,"x":6.134,"y":9.129,"output_tokens":1235,"input_tokens":370,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.108,"y":4.862,"output_tokens":1027,"input_tokens":37,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.909,"y":5.009,"output_tokens":652,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.948,"y":6.185,"output_tokens":10621,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":1,"x":4.983,"y":7.522,"output_tokens":158,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":1,"x":5.751,"y":7.442,"output_tokens":310,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":1,"x":6.093,"y":5.933,"output_tokens":315,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.812,"y":3.191,"output_tokens":128,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.306,"y":5.09,"output_tokens":356,"input_tokens":46,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.074,"y":4.951,"output_tokens":7872,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.054,"y":5.098,"output_tokens":380,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":2.438,"y":2.232,"output_tokens":60,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.402,"y":2.562,"output_tokens":173,"input_tokens":58,"token_source":"estimated","year":"2026"},{"group":-1,"x":2.529,"y":2.765,"output_tokens":240,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":1.447,"y":8.476,"output_tokens":199,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-1,"x":8.077,"y":3.446,"output_tokens":9222,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":1,"x":5.575,"y":6.687,"output_tokens":247,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":1,"x":5.909,"y":7.12,"output_tokens":9297,"input_tokens":64,"token_source":"estimated","year":"2026"},{"group":1,"x":6.14,"y":2.873,"output_tokens":109,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":1,"x":6.85,"y":3.137,"output_tokens":70,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":1,"x":6.147,"y":2.85,"output_tokens":196,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":1,"x":5.768,"y":6.28,"output_tokens":36111,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":0,"x":6.492,"y":9.392,"output_tokens":112,"input_tokens":102,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.552,"y":5.873,"output_tokens":131,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":1,"x":6.765,"y":2.915,"output_tokens":7941,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":1,"x":5.522,"y":6.293,"output_tokens":21138,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.007,"y":2.404,"output_tokens":1920,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":1,"x":6.481,"y":2.926,"output_tokens":2628,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":3,"x":9.531,"y":1.897,"output_tokens":544,"input_tokens":21,"token_source":"estimated","year":"2026"},{"group":8,"x":2.969,"y":2.422,"output_tokens":1835,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":8,"x":3.119,"y":2.572,"output_tokens":1769,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":3,"x":8.306,"y":2.492,"output_tokens":436,"input_tokens":137,"token_source":"estimated","year":"2026"},{"group":3,"x":9.256,"y":2.243,"output_tokens":11518,"input_tokens":23,"token_source":"estimated","year":"2026"},{"group":3,"x":9.178,"y":1.933,"output_tokens":457,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.586,"y":5.296,"output_tokens":24,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":2,"x":8.211,"y":2.515,"output_tokens":22297,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.253,"y":2.523,"output_tokens":167,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.301,"y":8.866,"output_tokens":219,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":3,"x":9.239,"y":2.021,"output_tokens":540,"input_tokens":30,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.57,"y":5.275,"output_tokens":22,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":3,"x":9.275,"y":2.475,"output_tokens":5129,"input_tokens":58,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.507,"y":2.675,"output_tokens":6875,"input_tokens":38,"token_source":"estimated","year":"2026"},{"group":0,"x":6.338,"y":9.464,"output_tokens":332,"input_tokens":64,"token_source":"estimated","year":"2026"},{"group":1,"x":5.505,"y":7.771,"output_tokens":458,"input_tokens":35,"token_source":"estimated","year":"2026"},{"group":1,"x":6.255,"y":7.616,"output_tokens":549,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":1,"x":6.482,"y":7.569,"output_tokens":543,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.844,"y":6.733,"output_tokens":167,"input_tokens":21,"token_source":"estimated","year":"2026"},{"group":1,"x":5.185,"y":7.289,"output_tokens":457,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":1,"x":1.253,"y":8.729,"output_tokens":9736,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":1,"x":5.298,"y":7.337,"output_tokens":293,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":1,"x":5.466,"y":6.671,"output_tokens":426,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":1,"x":5.602,"y":5.764,"output_tokens":175,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":1,"x":5.144,"y":7.084,"output_tokens":9455,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.495,"y":3.768,"output_tokens":337,"input_tokens":28,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.085,"y":6.821,"output_tokens":9971,"input_tokens":50,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.663,"y":5.3,"output_tokens":394,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":1,"x":5.328,"y":6.888,"output_tokens":241,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.439,"y":8.126,"output_tokens":79,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":1,"x":5.07,"y":7.653,"output_tokens":314,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":1,"x":5.266,"y":6.557,"output_tokens":20618,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":1,"x":5.158,"y":6.535,"output_tokens":1804,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":1,"x":5.31,"y":6.679,"output_tokens":5532,"input_tokens":42,"token_source":"estimated","year":"2026"},{"group":1,"x":5.211,"y":7.541,"output_tokens":9032,"input_tokens":359,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.709,"y":3.924,"output_tokens":5919,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.547,"y":9.711,"output_tokens":121,"input_tokens":2,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.239,"y":2.922,"output_tokens":9978,"input_tokens":258,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.313,"y":2.058,"output_tokens":293,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":0,"x":6.265,"y":9.454,"output_tokens":261,"input_tokens":39,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.858,"y":2.124,"output_tokens":18556,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":9,"x":6.269,"y":2.829,"output_tokens":7337,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":3,"x":9.602,"y":2.14,"output_tokens":15916,"input_tokens":21,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.411,"y":3.321,"output_tokens":54596,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.466,"y":3.366,"output_tokens":37874,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.825,"y":4.262,"output_tokens":180,"input_tokens":3,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.374,"y":8.862,"output_tokens":2202,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.417,"y":2.266,"output_tokens":409,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":9,"x":6.598,"y":2.956,"output_tokens":508,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.889,"y":2.141,"output_tokens":5334,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":8,"x":3.285,"y":2.026,"output_tokens":1572,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.324,"y":1.952,"output_tokens":452,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":1,"x":5.759,"y":5.951,"output_tokens":32,"input_tokens":26,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.152,"y":1.843,"output_tokens":317,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":8,"x":2.951,"y":2.175,"output_tokens":1650,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":0,"x":6.304,"y":9.459,"output_tokens":303,"input_tokens":55,"token_source":"estimated","year":"2026"},{"group":9,"x":6.185,"y":2.844,"output_tokens":341105,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":6,"x":7.343,"y":5.684,"output_tokens":0,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":6,"x":7.361,"y":5.741,"output_tokens":0,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":6,"x":7.315,"y":5.661,"output_tokens":26718,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.433,"y":5.417,"output_tokens":351,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":9,"x":6.526,"y":2.847,"output_tokens":0,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":9,"x":6.37,"y":2.901,"output_tokens":364769,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":6,"x":7.426,"y":5.795,"output_tokens":0,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":1,"x":5.616,"y":7.735,"output_tokens":669,"input_tokens":61,"token_source":"estimated","year":"2026"},{"group":1,"x":5.901,"y":7.746,"output_tokens":571,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":1,"x":7.131,"y":8.224,"output_tokens":497,"input_tokens":41,"token_source":"estimated","year":"2026"},{"group":1,"x":6.908,"y":8.5,"output_tokens":305,"input_tokens":87,"token_source":"estimated","year":"2026"},{"group":0,"x":6.78,"y":8.926,"output_tokens":398,"input_tokens":90,"token_source":"estimated","year":"2026"},{"group":1,"x":7.222,"y":3.12,"output_tokens":565,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":1,"x":5.54,"y":5.233,"output_tokens":47,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":1,"x":7.347,"y":7.421,"output_tokens":45,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":1,"x":4.692,"y":5.566,"output_tokens":8,"input_tokens":3,"token_source":"estimated","year":"2026"},{"group":0,"x":6.516,"y":9.679,"output_tokens":7,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":1,"x":6.923,"y":8.508,"output_tokens":390,"input_tokens":116,"token_source":"estimated","year":"2026"},{"group":0,"x":6.547,"y":9.736,"output_tokens":461,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":1,"x":6.686,"y":8.068,"output_tokens":381,"input_tokens":32,"token_source":"estimated","year":"2026"},{"group":5,"x":4.173,"y":3.572,"output_tokens":14621,"input_tokens":79,"token_source":"estimated","year":"2026"},{"group":3,"x":9.388,"y":2.12,"output_tokens":442,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":5,"x":4.106,"y":3.605,"output_tokens":8713,"input_tokens":34,"token_source":"estimated","year":"2026"},{"group":5,"x":4.257,"y":3.681,"output_tokens":8273,"input_tokens":34,"token_source":"estimated","year":"2026"},{"group":5,"x":4.192,"y":3.566,"output_tokens":7904,"input_tokens":33,"token_source":"estimated","year":"2026"},{"group":5,"x":4.291,"y":3.584,"output_tokens":494,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.033,"y":3.676,"output_tokens":368,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":5,"x":4.17,"y":3.584,"output_tokens":530,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.876,"y":3.918,"output_tokens":7459,"input_tokens":27,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.914,"y":3.539,"output_tokens":421,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.835,"y":3.617,"output_tokens":224,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":0,"x":6.161,"y":9.2,"output_tokens":341,"input_tokens":52,"token_source":"estimated","year":"2026"},{"group":5,"x":4.173,"y":3.596,"output_tokens":26145,"input_tokens":97,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.305,"y":5.859,"output_tokens":580,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.173,"y":6.426,"output_tokens":319,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.075,"y":6.32,"output_tokens":478,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.469,"y":2.141,"output_tokens":7381,"input_tokens":66,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.35,"y":2.86,"output_tokens":433,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.993,"y":3.819,"output_tokens":1685,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":3,"x":9.212,"y":2.551,"output_tokens":22572,"input_tokens":43,"token_source":"estimated","year":"2026"},{"group":0,"x":6.163,"y":9.342,"output_tokens":399,"input_tokens":78,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.072,"y":4.859,"output_tokens":107,"input_tokens":147,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.019,"y":4.753,"output_tokens":8113,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.204,"y":2.65,"output_tokens":2730,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.411,"y":2.053,"output_tokens":1976,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-2,"x":4.925,"y":5.276,"output_tokens":3441,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.024,"y":3.176,"output_tokens":8450,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":4.529,"y":4.764,"output_tokens":3458,"input_tokens":85,"token_source":"estimated","year":"2026"},{"group":-2,"x":9.031,"y":2.392,"output_tokens":27,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.102,"y":2.823,"output_tokens":553,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":3,"x":9.359,"y":2.392,"output_tokens":254,"input_tokens":175,"token_source":"estimated","year":"2026"},{"group":3,"x":8.997,"y":2.264,"output_tokens":304,"input_tokens":34,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.955,"y":2.234,"output_tokens":9402,"input_tokens":22,"token_source":"estimated","year":"2026"},{"group":-2,"x":4.583,"y":4.865,"output_tokens":131,"input_tokens":50,"token_source":"estimated","year":"2026"},{"group":0,"x":6.255,"y":9.536,"output_tokens":240,"input_tokens":32,"token_source":"estimated","year":"2026"},{"group":1,"x":5.252,"y":7.512,"output_tokens":393,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":1,"x":4.811,"y":6.667,"output_tokens":210,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":1,"x":4.696,"y":6.721,"output_tokens":7711,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":0,"x":5.157,"y":7.477,"output_tokens":11724,"input_tokens":107,"token_source":"estimated","year":"2026"},{"group":1,"x":4.999,"y":7.556,"output_tokens":3883,"input_tokens":28,"token_source":"estimated","year":"2026"},{"group":1,"x":5.034,"y":7.394,"output_tokens":2807,"input_tokens":38,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.575,"y":5.721,"output_tokens":441,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.443,"y":5.422,"output_tokens":685,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.112,"y":6.158,"output_tokens":19335,"input_tokens":59,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.852,"y":3.337,"output_tokens":14309,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.372,"y":6.291,"output_tokens":14831,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":6,"x":7.427,"y":5.767,"output_tokens":50,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.047,"y":4.888,"output_tokens":10221,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":5,"x":5.226,"y":3.993,"output_tokens":6965,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":9.383,"y":2.396,"output_tokens":5687,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":1,"x":5.273,"y":7.61,"output_tokens":509,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.364,"y":3.429,"output_tokens":5886,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":4,"x":3.242,"y":6.002,"output_tokens":449,"input_tokens":38,"token_source":"estimated","year":"2026"},{"group":4,"x":3.358,"y":6.03,"output_tokens":896,"input_tokens":68,"token_source":"estimated","year":"2026"},{"group":4,"x":3.198,"y":5.789,"output_tokens":748,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":4,"x":3.314,"y":5.68,"output_tokens":1389,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.459,"y":6.182,"output_tokens":853,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.719,"y":5.747,"output_tokens":779,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.55,"y":6.248,"output_tokens":607,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.324,"y":6.063,"output_tokens":925,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.185,"y":8.787,"output_tokens":1002,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-2,"x":4.097,"y":6.171,"output_tokens":143,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.511,"y":6.07,"output_tokens":185,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.685,"y":6.147,"output_tokens":1596,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.16,"y":6.365,"output_tokens":473,"input_tokens":137,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.196,"y":8.815,"output_tokens":121,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.004,"y":3.992,"output_tokens":498,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":2,"x":7.069,"y":2.171,"output_tokens":356,"input_tokens":34,"token_source":"estimated","year":"2026"},{"group":2,"x":7.184,"y":1.989,"output_tokens":851,"input_tokens":58,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.887,"y":3.537,"output_tokens":818,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.776,"y":2.961,"output_tokens":196,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.603,"y":3.397,"output_tokens":1530,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":8.279,"y":3.181,"output_tokens":2958,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":8.382,"y":3.404,"output_tokens":559,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":8.594,"y":3.564,"output_tokens":136,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-1,"x":8.25,"y":3.607,"output_tokens":23425,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":8.531,"y":3.458,"output_tokens":5071,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-1,"x":3.807,"y":5.498,"output_tokens":473,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.044,"y":5.762,"output_tokens":3585,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.653,"y":6.177,"output_tokens":273,"input_tokens":38,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.522,"y":6.3,"output_tokens":903,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.584,"y":6.211,"output_tokens":878,"input_tokens":28,"token_source":"estimated","year":"2026"},{"group":4,"x":3.379,"y":6.167,"output_tokens":863,"input_tokens":31,"token_source":"estimated","year":"2026"},{"group":4,"x":3.038,"y":5.838,"output_tokens":91,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.902,"y":5.902,"output_tokens":1244,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.063,"y":3.983,"output_tokens":311,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":0,"x":6.34,"y":9.636,"output_tokens":216,"input_tokens":35,"token_source":"estimated","year":"2026"},{"group":1,"x":6.066,"y":9.263,"output_tokens":243,"input_tokens":40,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.538,"y":7.183,"output_tokens":390,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":0,"x":6.3,"y":9.516,"output_tokens":258,"input_tokens":32,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.9,"y":2.009,"output_tokens":3186,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.877,"y":2.096,"output_tokens":255,"input_tokens":2,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.81,"y":2.071,"output_tokens":300,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":3,"x":9.303,"y":2.141,"output_tokens":6629,"input_tokens":313,"token_source":"estimated","year":"2026"},{"group":3,"x":9.43,"y":1.941,"output_tokens":7657,"input_tokens":165,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.176,"y":2.6,"output_tokens":7102,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.266,"y":3.192,"output_tokens":23,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.1,"y":2.89,"output_tokens":10348,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":4,"x":3.138,"y":5.9,"output_tokens":3511,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":0,"x":6.393,"y":9.533,"output_tokens":350,"input_tokens":54,"token_source":"estimated","year":"2026"},{"group":1,"x":6.233,"y":8.389,"output_tokens":1031,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":1,"x":4.763,"y":6.114,"output_tokens":528,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":1,"x":6.169,"y":8.441,"output_tokens":349,"input_tokens":35,"token_source":"estimated","year":"2026"},{"group":1,"x":6.568,"y":7.906,"output_tokens":352,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":1,"x":1.233,"y":8.735,"output_tokens":1323,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":1,"x":6.422,"y":7.542,"output_tokens":1141,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":1,"x":6.301,"y":7.944,"output_tokens":1018,"input_tokens":35,"token_source":"estimated","year":"2026"},{"group":1,"x":6.424,"y":7.838,"output_tokens":1053,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":1,"x":6.21,"y":8.024,"output_tokens":757,"input_tokens":40,"token_source":"estimated","year":"2026"},{"group":1,"x":6.22,"y":8.213,"output_tokens":723,"input_tokens":73,"token_source":"estimated","year":"2026"},{"group":1,"x":4.75,"y":5.454,"output_tokens":300,"input_tokens":3,"token_source":"estimated","year":"2026"},{"group":1,"x":6.454,"y":6.44,"output_tokens":269,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":1,"x":6.61,"y":7.375,"output_tokens":360,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":1,"x":6.384,"y":8.351,"output_tokens":420,"input_tokens":107,"token_source":"estimated","year":"2026"},{"group":1,"x":6.307,"y":8.351,"output_tokens":354,"input_tokens":73,"token_source":"estimated","year":"2026"},{"group":1,"x":6.821,"y":8.462,"output_tokens":473,"input_tokens":24,"token_source":"estimated","year":"2026"},{"group":0,"x":6.562,"y":9.749,"output_tokens":0,"input_tokens":3,"token_source":"estimated","year":"2026"},{"group":0,"x":6.423,"y":8.5,"output_tokens":451,"input_tokens":115,"token_source":"estimated","year":"2026"},{"group":1,"x":6.806,"y":8.109,"output_tokens":269,"input_tokens":28,"token_source":"estimated","year":"2026"},{"group":1,"x":6.514,"y":5.922,"output_tokens":340,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":1,"x":7.202,"y":5.926,"output_tokens":305,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":1,"x":7.001,"y":8.373,"output_tokens":268,"input_tokens":26,"token_source":"estimated","year":"2026"},{"group":1,"x":6.015,"y":8.031,"output_tokens":323,"input_tokens":56,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.792,"y":6.894,"output_tokens":93,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-1,"x":8.789,"y":1.942,"output_tokens":283,"input_tokens":12,"token_source":"estimated","year":"2026"},{"group":1,"x":6.183,"y":8.077,"output_tokens":1283,"input_tokens":86,"token_source":"estimated","year":"2026"},{"group":1,"x":6.413,"y":7.973,"output_tokens":1180,"input_tokens":76,"token_source":"estimated","year":"2026"},{"group":1,"x":6.595,"y":7.473,"output_tokens":1169,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":1,"x":6.049,"y":7.883,"output_tokens":1224,"input_tokens":30,"token_source":"estimated","year":"2026"},{"group":1,"x":5.883,"y":7.823,"output_tokens":1365,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":1,"x":6.402,"y":7.943,"output_tokens":1397,"input_tokens":24,"token_source":"estimated","year":"2026"},{"group":1,"x":6.147,"y":7.652,"output_tokens":1456,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":1,"x":6.003,"y":8.206,"output_tokens":1644,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":1,"x":6.169,"y":7.738,"output_tokens":632,"input_tokens":47,"token_source":"estimated","year":"2026"},{"group":1,"x":6.593,"y":7.774,"output_tokens":452,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":1,"x":7.146,"y":3.132,"output_tokens":1816,"input_tokens":5,"token_source":"estimated","year":"2026"},{"group":1,"x":4.808,"y":5.426,"output_tokens":738,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.541,"y":5.35,"output_tokens":264,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.178,"y":3.125,"output_tokens":872,"input_tokens":2,"token_source":"exact","year":"2026"},{"group":-1,"x":7.359,"y":7.996,"output_tokens":502,"input_tokens":38,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.296,"y":7.943,"output_tokens":509,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.317,"y":7.614,"output_tokens":469,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.853,"y":8.138,"output_tokens":488,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.261,"y":7.878,"output_tokens":476,"input_tokens":120,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.336,"y":7.992,"output_tokens":472,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.463,"y":7.812,"output_tokens":379,"input_tokens":40,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.528,"y":7.764,"output_tokens":437,"input_tokens":25,"token_source":"estimated","year":"2026"},{"group":-1,"x":2.398,"y":2.137,"output_tokens":233,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.607,"y":2.408,"output_tokens":2125,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.37,"y":2.487,"output_tokens":2396,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.09,"y":1.58,"output_tokens":187,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.16,"y":1.523,"output_tokens":307,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":4,"x":3.058,"y":5.786,"output_tokens":3558,"input_tokens":15,"token_source":"estimated","year":"2026"},{"group":4,"x":3.051,"y":5.878,"output_tokens":849,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.684,"y":5.209,"output_tokens":291,"input_tokens":4,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.651,"y":2.301,"output_tokens":2977,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.027,"y":3.234,"output_tokens":257,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-2,"x":2.632,"y":2.133,"output_tokens":202,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.98,"y":3.244,"output_tokens":26751,"input_tokens":14,"token_source":"estimated","year":"2026"},{"group":4,"x":2.995,"y":5.817,"output_tokens":152,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.63,"y":6.253,"output_tokens":264,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-1,"x":5.907,"y":4.033,"output_tokens":754,"input_tokens":19,"token_source":"estimated","year":"2026"},{"group":-2,"x":5.891,"y":4.836,"output_tokens":412,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.676,"y":3.462,"output_tokens":512,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.664,"y":3.224,"output_tokens":3561,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.108,"y":1.532,"output_tokens":284,"input_tokens":72,"token_source":"estimated","year":"2026"},{"group":-2,"x":3.161,"y":1.52,"output_tokens":240,"input_tokens":102,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.132,"y":2.755,"output_tokens":1993,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":6.016,"y":4.797,"output_tokens":942,"input_tokens":30,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.241,"y":8.753,"output_tokens":7266,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.415,"y":1.868,"output_tokens":12915,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":6.353,"y":1.857,"output_tokens":251,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":6.719,"y":2.48,"output_tokens":7384,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-1,"x":6.12,"y":1.928,"output_tokens":191,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-1,"x":6.592,"y":1.928,"output_tokens":6323,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":5.964,"y":2.07,"output_tokens":6197,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":6.578,"y":1.813,"output_tokens":1472,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":7.149,"y":1.815,"output_tokens":4319,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":7.736,"y":3.116,"output_tokens":586,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":7.527,"y":2.97,"output_tokens":2086,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":7.379,"y":3.457,"output_tokens":1625,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":7.413,"y":2.467,"output_tokens":3904,"input_tokens":3,"token_source":"exact","year":"2026"},{"group":-2,"x":8.426,"y":3.269,"output_tokens":73870,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.135,"y":2.873,"output_tokens":147,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-2,"x":1.194,"y":8.715,"output_tokens":4898,"input_tokens":1,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.845,"y":2.098,"output_tokens":182,"input_tokens":10,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.809,"y":2.006,"output_tokens":1567,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.941,"y":2.301,"output_tokens":4208,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":8.596,"y":2.358,"output_tokens":5317,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":9.561,"y":2.409,"output_tokens":330,"input_tokens":16,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.337,"y":5.446,"output_tokens":300,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-1,"x":7.491,"y":4.1,"output_tokens":439,"input_tokens":3,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.205,"y":5.433,"output_tokens":362,"input_tokens":9,"token_source":"estimated","year":"2026"},{"group":6,"x":6.663,"y":5.593,"output_tokens":8065,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.355,"y":5.675,"output_tokens":746,"input_tokens":11,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.251,"y":3.192,"output_tokens":571,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":3,"x":9.434,"y":2.511,"output_tokens":8398,"input_tokens":26,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.54,"y":4.895,"output_tokens":2509,"input_tokens":8,"token_source":"estimated","year":"2026"},{"group":-1,"x":6.39,"y":6.134,"output_tokens":7489,"input_tokens":18,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.631,"y":4.928,"output_tokens":146,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":3,"x":9.591,"y":2.329,"output_tokens":34,"input_tokens":17,"token_source":"estimated","year":"2026"},{"group":-1,"x":4.821,"y":6.568,"output_tokens":118,"input_tokens":7,"token_source":"estimated","year":"2026"},{"group":-1,"x":9.509,"y":2.012,"output_tokens":5570,"input_tokens":20,"token_source":"estimated","year":"2026"},{"group":-1,"x":3.586,"y":3.105,"output_tokens":6809,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.171,"y":5.533,"output_tokens":646,"input_tokens":21,"token_source":"estimated","year":"2026"},{"group":-2,"x":6.933,"y":2.825,"output_tokens":3435,"input_tokens":13,"token_source":"estimated","year":"2026"},{"group":-2,"x":7.626,"y":2.398,"output_tokens":957,"input_tokens":6,"token_source":"estimated","year":"2026"},{"group":3,"x":9.456,"y":1.924,"output_tokens":10130,"input_tokens":22,"token_source":"estimated","year":"2026"}],"groups":[{"id":1,"label":"Team coaching & sports planning","n":97,"input":2981,"output":246227},{"id":3,"label":"Travel itinerary planning","n":21,"input":1220,"output":141104},{"id":0,"label":"Translation requests","n":19,"input":1500,"output":17428},{"id":2,"label":"Calendar & scheduling","n":17,"input":272,"output":53409},{"id":4,"label":"Music / chord progression","n":11,"input":237,"output":13077},{"id":5,"label":"Shopping research","n":10,"input":340,"output":83183},{"id":6,"label":"Presentation & slide content","n":10,"input":169,"output":64018},{"id":8,"label":"Recipes & cooking","n":9,"input":88,"output":7467},{"id":9,"label":"Google Drive documents","n":7,"input":42,"output":713854},{"id":-2,"label":"Smaller clusters","n":143,"input":4068,"output":556606},{"id":-1,"label":"Unclustered","n":119,"input":2306,"output":318101}],"n_communities":80,"total_prompts":463}</script>
<script>
(function() {
  const root = document.getElementById('cluster-viz');
  const DATA = JSON.parse(document.getElementById('cluster-viz-data').textContent);
  const points = DATA.points;
  const groups = DATA.groups;

  const COLORS = ['#2a78d6', '#1baf7a', '#eda100', '#008300', '#4a3aa7', '#e34948', '#e87ba4', '#eb6834', '#0099a8'];
  const COLOR_OTHER = '#c3c2b7';
  const COLOR_SINGLE = '#d9d8d0';
  const TOP8_IDS = groups.filter(g => g.id >= 0).map(g => g.id);

  function groupColor(gid) {
    if (gid === -1) return COLOR_SINGLE;
    if (gid === -2) return COLOR_OTHER;
    const idx = TOP8_IDS.indexOf(gid);
    return idx >= 0 ? COLORS[idx] : COLOR_OTHER;
  }
  function groupLabel(gid) {
    const g = groups.find(x => x.id === gid);
    return g ? g.label : 'Other';
  }

  root.innerHTML = `
    <div class="stat-row" id="stats"></div>
    <div class="chart-wrap"><canvas id="scatter"></canvas></div>
    <div class="legend" id="legend"></div>
    <div class="cost-wrap">
      <h2>Tokens &amp; est. cost by group</h2>
      <div id="cost-table"></div>
      <p class="cost-note">Rates: $3.00 input / $15.00 output per million tokens. Web-app rows use estimated token counts.</p>
    </div>
  `;

  const stats = root.querySelector('#stats');
  const n_clustered = points.filter(p => p.group !== -1).length;
  const avg_out = Math.round(points.reduce((s, p) => s + p.output_tokens, 0) / points.length);
  stats.innerHTML = `
    <div class="stat"><span class="stat-val">${DATA.total_prompts}</span><span class="stat-lbl">Prompts</span></div>
    <div class="stat"><span class="stat-val">${DATA.n_communities}</span><span class="stat-lbl">Communities</span></div>
    <div class="stat"><span class="stat-val">${n_clustered}</span><span class="stat-lbl">In a community</span></div>
    <div class="stat"><span class="stat-val">${avg_out.toLocaleString()}</span><span class="stat-lbl">Avg output tokens</span></div>
  `;

  const canvas = root.querySelector('#scatter');
  const ctx = canvas.getContext('2d');
  let tooltip = document.getElementById('cluster-viz-tooltip');
  if (!tooltip) {
    tooltip = document.createElement('div');
    tooltip.id = 'cluster-viz-tooltip';
    tooltip.setAttribute('id', 'tooltip');
    document.body.appendChild(tooltip);
  }

  const xs = points.map(p => p.x), ys = points.map(p => p.y);
  const xMin = Math.min(...xs), xMax = Math.max(...xs);
  const yMin = Math.min(...ys), yMax = Math.max(...ys);
  const PAD = 0.5;
  let W, H, dpr;
  let activeFilters = new Set();

  function toCanvas(x, y) {
    const margin = 30;
    const cx = margin + (x - xMin + PAD) / (xMax - xMin + 2 * PAD) * (W - 2 * margin);
    const cy = H - margin - (y - yMin + PAD) / (yMax - yMin + 2 * PAD) * (H - 2 * margin);
    return [cx, cy];
  }
  function isVisible(p) {
    return activeFilters.size === 0 || activeFilters.has(p.group);
  }
  function resize() {
    dpr = window.devicePixelRatio || 1;
    const rect = canvas.getBoundingClientRect();
    W = rect.width; H = rect.height;
    canvas.width = W * dpr; canvas.height = H * dpr;
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.scale(dpr, dpr);
    draw();
  }
  function draw() {
    ctx.clearRect(0, 0, W, H);
    ctx.fillStyle = '#fcfcfb';
    ctx.fillRect(0, 0, W, H);
    ctx.strokeStyle = '#e1e0d9';
    ctx.lineWidth = 0.5;
    for (let gx = Math.ceil(xMin); gx <= Math.floor(xMax); gx++) {
      const [cx] = toCanvas(gx, yMin);
      ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, H); ctx.stroke();
    }
    for (let gy = Math.ceil(yMin); gy <= Math.floor(yMax); gy++) {
      const [, cy] = toCanvas(xMin, gy);
      ctx.beginPath(); ctx.moveTo(0, cy); ctx.lineTo(W, cy); ctx.stroke();
    }
    const singles = points.filter(p => p.group === -1);
    const rest = points.filter(p => p.group !== -1);
    function drawPoint(p, r) {
      const visible = isVisible(p);
      const [cx, cy] = toCanvas(p.x, p.y);
      ctx.beginPath();
      ctx.arc(cx, cy, r, 0, Math.PI * 2);
      ctx.fillStyle = groupColor(p.group);
      ctx.globalAlpha = visible ? (p.group === -1 ? 0.45 : 0.85) : 0.08;
      ctx.fill();
      ctx.globalAlpha = 1;
    }
    singles.forEach(p => drawPoint(p, 3.5));
    rest.forEach(p => drawPoint(p, 5));
  }

  let hovered = null;
  canvas.addEventListener('mousemove', e => {
    const rect = canvas.getBoundingClientRect();
    const mx = e.clientX - rect.left, my = e.clientY - rect.top;
    let best = null, bestDist = 14;
    for (const p of points) {
      if (!isVisible(p)) continue;
      const [cx, cy] = toCanvas(p.x, p.y);
      const dd = Math.hypot(mx - cx, my - cy);
      if (dd < bestDist) { bestDist = dd; best = p; }
    }
    if (best !== hovered) {
      hovered = best;
      draw();
      if (best) {
        const [cx, cy] = toCanvas(best.x, best.y);
        ctx.beginPath(); ctx.arc(cx, cy, 7, 0, Math.PI * 2);
        ctx.strokeStyle = '#14140f'; ctx.lineWidth = 1.5; ctx.stroke();
      }
    }
    if (best) {
      const label = groupLabel(best.group);
      tooltip.innerHTML = `
        <div class="tt-cat">${label}</div>
        <div class="tt-meta">
          ${best.year} &middot; output ${best.output_tokens.toLocaleString()} tok
          (${best.token_source === 'exact' ? 'exact' : 'est.'})
        </div>`;
      tooltip.style.display = 'block';
      const tx = e.clientX + 16, ty = e.clientY - 10;
      const tw = 280, th = 70;
      tooltip.style.left = (tx + tw > window.innerWidth ? e.clientX - tw - 8 : tx) + 'px';
      tooltip.style.top = (ty + th > window.innerHeight ? e.clientY - th : ty) + 'px';
    } else {
      tooltip.style.display = 'none';
    }
  });
  canvas.addEventListener('mouseleave', () => { tooltip.style.display = 'none'; hovered = null; draw(); });

  function buildLegend() {
    const legendEl = root.querySelector('#legend');
    legendEl.innerHTML = '';
    const items = groups.map(g => ({ id: g.id, label: g.label, color: groupColor(g.id), n: g.n }));
    items.forEach(item => {
      const el = document.createElement('div');
      el.className = 'legend-item' + (activeFilters.size > 0 && !activeFilters.has(item.id) ? ' dimmed' : '');
      el.innerHTML = `<span class="legend-dot" style="background:${item.color}"></span>${item.label} <span style="color:var(--ink-m)">(${item.n})</span>`;
      el.addEventListener('click', () => {
        if (activeFilters.has(item.id)) activeFilters.delete(item.id);
        else activeFilters.add(item.id);
        buildLegend();
        draw();
      });
      legendEl.appendChild(el);
    });
  }

  let sortCol = 'cost', sortDir = -1;
  function buildCostTable() {
    const rows = groups.map(g => {
      const cost = (g.input / 1e6) * 3.00 + (g.output / 1e6) * 15.00;
      return { id: g.id, label: g.label, n: g.n, input: g.input, output: g.output, cost, color: groupColor(g.id) };
    });
    rows.sort((a, b) => {
      const av = a[sortCol], bv = b[sortCol];
      return typeof av === 'string' ? sortDir * av.localeCompare(bv) : sortDir * (av - bv);
    });
    const cols = [
      { key: 'label', head: 'Group', cls: '' },
      { key: 'n', head: 'Prompts', cls: 'num' },
      { key: 'input', head: 'Input tok', cls: 'num' },
      { key: 'output', head: 'Output tok', cls: 'num' },
      { key: 'cost', head: 'Est. cost', cls: 'cost-val' },
    ];
    let html = `<table class="cost-table"><thead><tr>`;
    for (const col of cols) {
      const sc = sortCol === col.key ? `sort-${sortDir > 0 ? 'asc' : 'desc'}` : '';
      html += `<th class="${sc}" data-key="${col.key}">${col.head}</th>`;
    }
    html += `</tr></thead><tbody>`;
    for (const r of rows) {
      html += `<tr>
        <td><span class="swatch" style="background:${r.color}"></span>${r.label}</td>
        <td class="num">${r.n.toLocaleString()}</td>
        <td class="num">${r.input.toLocaleString()}</td>
        <td class="num">${r.output.toLocaleString()}</td>
        <td class="cost-val">$${r.cost < 0.01 ? r.cost.toFixed(4) : r.cost.toFixed(3)}</td>
      </tr>`;
    }
    const totIn = rows.reduce((s, r) => s + r.input, 0);
    const totOut = rows.reduce((s, r) => s + r.output, 0);
    const totCost = rows.reduce((s, r) => s + r.cost, 0);
    html += `<tr style="font-weight:600;border-top:2px solid var(--grid)">
      <td>Total</td>
      <td class="num">${DATA.total_prompts.toLocaleString()}</td>
      <td class="num">${totIn.toLocaleString()}</td>
      <td class="num">${totOut.toLocaleString()}</td>
      <td class="cost-val">$${totCost.toFixed(3)}</td>
    </tr>`;
    html += `</tbody></table>`;
    const el = root.querySelector('#cost-table');
    el.innerHTML = html;
    el.querySelectorAll('th').forEach(th => {
      th.addEventListener('click', () => {
        const key = th.dataset.key;
        if (sortCol === key) sortDir *= -1;
        else { sortCol = key; sortDir = key === 'label' ? 1 : -1; }
        buildCostTable();
      });
    });
  }

  window.addEventListener('resize', resize);
  buildLegend();
  buildCostTable();
  resize();
})();
</script>


## What It Shows, and What That Maps to at Enterprise Scale

A few things stood out once the communities were laid out, and each one has a direct analogue in the enterprise numbers above:

- **The single largest community wasn't technical at all** — youth sports team coaching, 97 prompts, more than double the next biggest group. Use-case clustering surfaces where volume actually concentrates, which is very often not where anyone assumed it would be. At enterprise scale, that's the difference between believing spend is driven by "engineering" and discovering it's actually concentrated in three or four specific repeated workflows, some of which have nothing to do with the team nominally paying for them.
- **A quarter of everything I've sent Claude never joined a community.** 119 of 463 prompts were similarity-orphans — one-off, novel requests. A growing unclustered fraction at enterprise scale cuts two ways: it can mean a genuinely new use case is emerging and worth investing in (good), or it can mean nobody has standardized how a given task gets asked, so the same job gets solved a slightly different, slightly more expensive way every time by every person who touches it (bad, and exactly the kind of thing that produces a $500 million surprise when nobody's watching the aggregate).
- **Cost and prompt count don't track together.** My biggest cluster by volume wasn't my biggest cluster by cost — smaller, chattier clusters rack up disproportionate output-token spend per prompt because the responses run long. This is the same disconnect Uber's own leadership pointed to: high token usage that doesn't correlate cleanly with anything being shipped. You cannot see this disconnect in a total-dollars number; you can only see it once spend is broken out by what generated it.
- **`exact` vs. `estimated` matters.** Claude Code's logged token counts and my tokenizer-based estimates for the web app agree in aggregate but diverge enough at the row level that I wouldn't trust the estimated rows for anything more precise than the cost table above. An enterprise system built on inconsistent metering across tools will inherit the same blind spot, just at a scale where it's a lot more expensive to be wrong.

### What $1,000/Day on a Single API Key Actually Looks Like

A useful way to pressure-test the clustering framework: take a single API key spending $1,000/day — not a team, not an org, one key — and work backwards to what must be generating that spend.

At Sonnet 4.6 pricing ($3 input / $15 output per million tokens), $1,000/day is roughly 66 million output tokens. That rules out a single heavy user immediately; even the most expensive cluster in my own data — seven short prompts that triggered full document generation — ran to about $11 total. To hit $1,000 per day you need either enormous volume, very long outputs, or both, running continuously.

Three scenarios account for almost everything you'll actually see:

**Always-on document processing.** Legal discovery, insurance claims, medical records, compliance review — pipelines that ingest long documents (2–5K input tokens each) and emit short structured outputs, running 24/7 without human intervention. Even 5,000 documents/day at 3K input tokens each pushes well past $1,000 without any single output being particularly large. The spend pattern is flat around the clock, because there's no human in the loop.

**A customer-facing product with real traction.** A B2B SaaS or consumer app where Claude powers a core feature — drafting, analysis, search — at scale. 20,000 multi-turn conversations/day with 4K of RAG context stuffed in per turn gets there fast. The signature here is spend that's spiky during business hours (B2B) or evenings (consumer), not flat — because users drive it, not a scheduler.

**An autonomous agent loop running continuously.** A single orchestrated system — research agent, coding agent, competitive intelligence pipeline — running long multi-step jobs. The killer cost driver isn't the individual step: it's that the full context window gets re-sent every step. One agent at 200 steps × 50K context = 10 million input tokens before counting output. Ten parallel agents hits $1,000 quickly, and most orgs running these haven't yet enabled prompt caching, which would cut effective input cost by an order of magnitude.

The practical monitoring implication: the *shape* of the spend over time is more diagnostic than the total. Flat 24/7 → pipeline. Business-hours spiky → product with users. Overnight bursts → agent jobs. A raw dollar total tells you none of this; attribution plus time-series anomaly detection tells you all of it.

## What a Real Monitoring System Needs

Scaling this from one person's laptop to an organization isn't a bigger version of the cost table — it's three specific pieces, and none of them is exotic on its own:

**An attribution layer.** Every request needs to be tagged with who generated it and why before any of the rest of this is possible — the $500 million bill only happened because nobody could see usage accumulating by employee in the first place. A lightweight proxy sitting in front of whichever model providers you use (routing every call through one gateway with consistent logging, rather than a scattered mix of direct API keys per team) gets you this almost for free, and it's also the natural place to enforce the kind of per-engineer budget Jensen Huang has publicly recommended.

**Semantic clustering for cost-center discovery, not just cost totals.** The same embed-and-threshold approach from the demo above works on an enterprise prompt log and answers a more useful question than "which team spent the most": *which use cases* are driving spend, regardless of which team happens to own them. If three departments have each independently built a cluster of prompts that all boil down to "summarize this document," that's not three cost centers — it's one automatable workflow and a caching opportunity that a team-by-team dashboard would never surface. This is exactly the structure my <a href="/community-detection-viz/">SBM/GNN community detection work</a> was built for; only what the nodes represent changes.

**Streaming anomaly detection on the cost signal itself, not a fixed dollar threshold.** A static alert ("page someone if spend exceeds $X/day") fires too late for a runaway agent loop and too often for a normal end-of-quarter spike — which is precisely how a company burns four months of AI budget in four months instead of catching it in week one. The <a href="/hypervector-anomaly-detection/">hyperdimensional streaming approach</a> I wrote about for network anomaly detection generalizes directly here: treat the stream of (team, model, token count) events the way HDSGA treats a stream of graph snapshots, encode it into constant-memory hypervectors, and flag statistical deviations from a learned baseline instead of a fixed number. An agent stuck in a retry loop looks a lot like a DDoS in token-space, and the same constant-memory, real-time properties that make HDC attractive for network security make it attractive here — you want the alert before the invoice, not after.

## From Principle to Practice: GitLab Templates and LiteLLM Guardrails

The attribution layer described above is the most important piece of the stack, and the one most routinely skipped — because the path of least resistance for any engineer is to paste an API key directly into their shell. The question is how to make the attributable path *easier* than the unattributable one, which means encoding the tagging into the place where engineering work actually starts: the git repository.

### Auto-populating `.claude/settings.json` via GitLab

GitLab's [group-level compliance pipeline](https://docs.gitlab.com/ee/user/group/compliance_frameworks.html) or a repository creation webhook can automatically commit a `.claude/settings.json` to every new repo the moment it's initialized. Claude Code reads this file before making any API call; its `env` block sets environment variables for every session opened against that repo:

```json
{
  "env": {
    "CLAUDE_PROJECT": "{% raw %}{{ project_name }}{% endraw %}",
    "CLAUDE_TEAM":    "{% raw %}{{ gitlab_namespace }}{% endraw %}",
    "CLAUDE_CC":      "{% raw %}{{ cost_center_tag }}{% endraw %}"
  },
  "apiBaseUrl": "https://llm-proxy.internal/v1"
}
```

The `apiBaseUrl` field is the critical part: it points Claude Code at the organization's LiteLLM proxy instead of directly at the Anthropic API. Engineers get a key scoped to the proxy, not a raw Anthropic key, so there's no way to accidentally (or deliberately) bypass attribution. The `.gitlab-ci.yml` template that creates the file is straightforward:

```yaml
# .gitlab/ci/ensure-claude-settings.yml  (included in group-level compliance pipeline)
ensure-claude-settings:
  stage: setup
  rules:
    - if: '$CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      changes:
        - .claude/settings.json
      when: never     # skip if the file already exists and is being updated intentionally
    - if: '$CI_PIPELINE_SOURCE == "push" && $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
  script:
    - |
      mkdir -p .claude
      cat > .claude/settings.json <<EOF
      {
        "env": {
          "CLAUDE_PROJECT": "$CI_PROJECT_NAME",
          "CLAUDE_TEAM":    "$CI_PROJECT_NAMESPACE",
          "CLAUDE_CC":      "$BILLING_COST_CENTER"
        },
        "apiBaseUrl": "https://llm-proxy.internal/v1"
      }
      EOF
      git config user.email "ci-bot@company.internal"
      git config user.name  "Compliance Bot"
      git add .claude/settings.json
      git diff --cached --quiet || git commit -m "chore: add .claude/settings.json [skip ci]"
      git push origin HEAD:$CI_COMMIT_BRANCH
```

`BILLING_COST_CENTER` gets injected from the GitLab group's CI/CD variable settings, so the right cost center follows the namespace without anyone having to think about it.

### Blocking expensive requests with LiteLLM guardrails

LiteLLM's [custom guardrail](https://docs.litellm.ai/docs/proxy/guardrails/custom_guardrail) hooks run synchronously in the request path and can reject before a token is sent. A pre-call hook that enforces attribution and daily budget is roughly 40 lines:

```python
# guardrails/attribution.py
from litellm.integrations.custom_guardrail import CustomGuardrail
from litellm import ModelResponse
from datetime import date
import redis, os

r = redis.Redis.from_url(os.environ["REDIS_URL"])

DAILY_TOKEN_LIMITS = {
    "platform-eng": 10_000_000,
    "default":         500_000,
}
MAX_REQUEST_OUTPUT_TOKENS = 16_000   # hard cap per call


class AttributionGuardrail(CustomGuardrail):
    async def async_pre_call_hook(self, user_api_key_dict, cache, data, call_type):
        meta = data.get("metadata", {})
        team = meta.get("team") or _header(data, "x-claude-team")
        cc   = meta.get("cost_center") or _header(data, "x-claude-cc")

        if not team or not cc:
            raise PermissionError(
                "Requests must include metadata.team and metadata.cost_center. "
                "Ensure .claude/settings.json is committed to your repo."
            )

        # Enforce per-request output cap
        params = data.get("parameters", {})
        if params.get("max_tokens", 0) > MAX_REQUEST_OUTPUT_TOKENS:
            raise ValueError(
                f"max_tokens {params['max_tokens']} exceeds org limit "
                f"of {MAX_REQUEST_OUTPUT_TOKENS}"
            )

        # Check daily budget
        key = f"tokens:{team}:{date.today()}"
        used = int(r.get(key) or 0)
        limit = DAILY_TOKEN_LIMITS.get(team, DAILY_TOKEN_LIMITS["default"])
        if used >= limit:
            raise PermissionError(
                f"Team '{team}' has exhausted its daily token budget "
                f"({limit:,} tokens). Contact #platform-eng to adjust."
            )

        return data

    async def async_post_call_success_hook(self, data, user_api_key_dict, response):
        team = data.get("metadata", {}).get("team") or _header(data, "x-claude-team")
        if team and hasattr(response, "usage"):
            key = f"tokens:{team}:{date.today()}"
            r.incrby(key, response.usage.total_tokens)
            r.expire(key, 86_400 * 2)   # 2-day TTL for cleanup


def _header(data, name):
    return (data.get("headers") or {}).get(name)
```

Wire it into `litellm_settings` in your proxy config:

```yaml
# config.yaml
litellm_settings:
  success_callback: ["langfuse"]     # or whatever your logging backend is
  guardrails:
    - guardrail_name: "attribution"
      litellm_params:
        guardrail: guardrails.attribution.AttributionGuardrail
        mode: "pre_call_and_during_call"
```

What this gets you end-to-end: engineers can't bypass attribution because the only keys they can obtain route through the proxy. The `.claude/settings.json` committed to every repo provides the metadata automatically, so compliance friction approaches zero. The proxy sees every request, which is exactly the event stream the clustering and anomaly detection run on.

One gap worth naming: this architecture covers Claude Code traffic (where the settings file is respected), but engineers calling the API directly from application code need to pass `metadata.team` and `metadata.cost_center` explicitly, or be given keys scoped to a proxy route that injects defaults for them. LiteLLM supports key-level metadata for exactly this — a per-team API key can have a `metadata` block baked in at issuance, so application code doesn't need to know about cost centers at all. That's a policy decision, not a technical constraint.

### Classifying requests by intent

Budget caps and attribution checks handle the *volume* side of runaway spend; they don't catch *intent*: a developer pointing an agent loop at Google Drive and asking Claude to create documents for every meeting note, bypassing the company's document management workflow. A lightweight intent classifier in the same pre-call hook catches these before a single token is sent.

The approach reuses `all-MiniLM-L6-v2` — the same model behind the clustering above. Embed the incoming request, compare it against precomputed centroids for each blocked category, block if cosine similarity exceeds a threshold. No labeled training data, no separate service to operate, easy to extend by adding example phrases:

```python
# guardrails/classifier.py
from sentence_transformers import SentenceTransformer
import numpy as np

BLOCKED: dict[str, dict] = {
    "google_doc_creation": {
        "examples": [
            "create a Google Doc",
            "make a new Google document",
            "write this to Google Docs",
            "save this to Drive as a document",
            "generate a Google Doc from this",
            "open a new doc in Drive",
            "create a document in my Google Drive",
        ],
        "threshold": 0.70,
        "reply": (
            "Creating Google Docs is handled by the document workflow service. "
            "Use /workflows/doc-create instead of asking Claude directly."
        ),
    },
    # "bulk_email_draft", "pii_extraction", etc. follow the same pattern
}


class RequestClassifier:
    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.model = SentenceTransformer(model_name)   # 22 MB, loads once at startup
        self._centroids = self._build_centroids()

    def _build_centroids(self) -> dict:
        out = {}
        for name, cfg in BLOCKED.items():
            vecs = self.model.encode(cfg["examples"], normalize_embeddings=True)
            centroid = vecs.mean(axis=0)
            centroid /= np.linalg.norm(centroid)        # re-normalize after averaging
            out[name] = {
                "centroid": centroid,
                "threshold": cfg["threshold"],
                "reply": cfg["reply"],
            }
        return out

    def classify(self, text: str) -> tuple:
        """Returns (category, similarity) if blocked, else (None, 0.0)."""
        vec = self.model.encode([text], normalize_embeddings=True)[0]
        best_name, best_sim = None, 0.0
        for name, data in self._centroids.items():
            sim = float(np.dot(vec, data["centroid"]))  # cosine sim (both normalized)
            if sim >= data["threshold"] and sim > best_sim:
                best_name, best_sim = name, sim
        return best_name, best_sim
```

Wire it into `AttributionGuardrail` with three additions — instantiate it in `__init__`, extract the last user message, and check it before the request is forwarded:

```python
# In AttributionGuardrail.__init__:
from guardrails.classifier import RequestClassifier, BLOCKED
self.classifier = RequestClassifier()

# At the end of async_pre_call_hook, after the budget check:
last_user = next(
    (m["content"] for m in reversed(data.get("messages", []))
     if isinstance(m.get("content"), str) and m.get("role") == "user"),
    "",
)
if last_user:
    category, score = self.classifier.classify(last_user)
    if category:
        raise PermissionError(
            f"[{category} score={score:.2f}] {BLOCKED[category]['reply']}"
        )
```

A few calibration notes worth keeping in mind before deploying:

- **Threshold 0.70** is a conservative starting point. Log `score` on every request for a few days to inspect the distribution before tightening. Lower the threshold if legitimate requests are slipping through; raise it if edge-case legitimate prompts are being blocked.
- **`normalize_embeddings=True`** in the `encode` call means `np.dot` equals cosine similarity — no division needed at inference time.
- **Adding examples shifts the centroid, not the model.** If a paraphrase slips through, add it to the `examples` list and restart the proxy. No retraining required.
- **Inference speed is negligible.** `all-MiniLM-L6-v2` runs `classify` in under 5 ms on a single CPU core. If you're on a very constrained host, `paraphrase-MiniLM-L3-v2` is half the size with a modest accuracy trade-off.

The semantic approach catches paraphrasing that keyword lists miss. "Write this up as a Doc", "Drop it into Drive", and "Can you spin up a Google document for this?" all land above 0.70 for `google_doc_creation` without any of those exact strings appearing in the example set. That robustness matters in practice: the requests you want to block rarely arrive with the exact phrasing you anticipated.

## The Actual Point

The CEOs described as "baffled" by AI costs aren't wrong that the bills are large. They're working from a monitoring setup that was never going to answer the question they're actually asking, which isn't "how much did we spend" but "was that spend doing anything, and would we have known if it wasn't." Tokens-times-rate answers the first question and is silent on the second. Attribution, use-case clustering, and streaming anomaly detection together answer both — and none of the three requires anything more exotic than the embedding-and-graph toolkit already sitting behind most of this blog. The gap between where most organizations are and where they need to be isn't a technology gap. It's that almost nobody has built the second half of the pipeline yet.

The code for the embedding, clustering, and UMAP pipeline behind the demo above is a short, self-contained script — nothing in it needs more than `sentence-transformers`, `umap-learn`, and a CSV of your own history if you want to see what your own usage clusters into.
