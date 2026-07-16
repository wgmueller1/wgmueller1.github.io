---
layout: post
title: Differentiable Community Detection with GNNs and Stochastic Block Models
description: "Using SBM likelihood as loss functions for unsupervised GNN training"
modified: 2025-12-28
tags: [graph,neural,networks,community,detection,stochastic,block,models,machine,learning]
comments: true
published: true
image:
  feature:
  credit:
  creditlink:
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

Community detection is a fundamental problem in network analysis - identifying groups of nodes that are more densely connected internally than to the rest of the network. In our recent <a href="https://openreview.net/forum?id=T1vdfm1THf" target="_blank">paper</a> presented at the Learning on Graphs Conference (LoG 2025), we propose a novel approach that combines Graph Neural Networks (GNNs) with Stochastic Block Models (SBMs) to create a differentiable, architecture-agnostic framework for community detection.


<div id="graph"></div>

<style>
#graph {
  width: 100%;
  height: 300px;
  border: none;
  margin: 20px 0;
}

.node {
  stroke: #fff;
  stroke-width: 2px;
  cursor: pointer;
}

.link {
  stroke: #999;
  stroke-opacity: 0.6;
}

.node:hover {
  stroke: #000;
  stroke-width: 3px;
}

.node-label {
  font-size: 10px;
  pointer-events: none;
  text-anchor: middle;
}
</style>

<script src="https://d3js.org/d3.v7.min.js"></script>
<script>
// Network with three communities
const data = {
  nodes: [
    // Community 0 (blue) - nodes 0-9
    {id: 0, community: 0}, {id: 1, community: 0}, {id: 2, community: 0}, {id: 3, community: 0},
    {id: 4, community: 0}, {id: 5, community: 0}, {id: 6, community: 0}, {id: 7, community: 0},
    {id: 8, community: 0}, {id: 9, community: 0},
    // Community 1 (orange) - nodes 10-19
    {id: 10, community: 1}, {id: 11, community: 1}, {id: 12, community: 1}, {id: 13, community: 1},
    {id: 14, community: 1}, {id: 15, community: 1}, {id: 16, community: 1}, {id: 17, community: 1},
    {id: 18, community: 1}, {id: 19, community: 1},
    // Community 2 (green) - nodes 20-29
    {id: 20, community: 2}, {id: 21, community: 2}, {id: 22, community: 2}, {id: 23, community: 2},
    {id: 24, community: 2}, {id: 25, community: 2}, {id: 26, community: 2}, {id: 27, community: 2},
    {id: 28, community: 2}, {id: 29, community: 2}
  ],
  links: [
    // Community 0 internal edges
    {source: 0, target: 1}, {source: 0, target: 2}, {source: 0, target: 3}, {source: 0, target: 4},
    {source: 1, target: 2}, {source: 1, target: 3}, {source: 1, target: 5}, {source: 2, target: 3},
    {source: 2, target: 6}, {source: 3, target: 4}, {source: 3, target: 7}, {source: 4, target: 5},
    {source: 5, target: 6}, {source: 5, target: 8}, {source: 6, target: 7}, {source: 7, target: 8},
    {source: 7, target: 9}, {source: 8, target: 9},

    // Community 1 internal edges
    {source: 10, target: 11}, {source: 10, target: 12}, {source: 10, target: 13}, {source: 10, target: 14},
    {source: 11, target: 12}, {source: 11, target: 13}, {source: 11, target: 15}, {source: 12, target: 13},
    {source: 12, target: 16}, {source: 13, target: 14}, {source: 13, target: 17}, {source: 14, target: 15},
    {source: 15, target: 16}, {source: 15, target: 18}, {source: 16, target: 17}, {source: 17, target: 18},
    {source: 17, target: 19}, {source: 18, target: 19},

    // Community 2 internal edges
    {source: 20, target: 21}, {source: 20, target: 22}, {source: 20, target: 23}, {source: 20, target: 24},
    {source: 21, target: 22}, {source: 21, target: 23}, {source: 21, target: 25}, {source: 22, target: 23},
    {source: 22, target: 26}, {source: 23, target: 24}, {source: 23, target: 27}, {source: 24, target: 25},
    {source: 25, target: 26}, {source: 25, target: 28}, {source: 26, target: 27}, {source: 27, target: 28},
    {source: 27, target: 29}, {source: 28, target: 29},

    // Inter-community edges (bridge connections)
    {source: 4, target: 10}, {source: 9, target: 15}, {source: 6, target: 11},
    {source: 14, target: 20}, {source: 19, target: 25}, {source: 16, target: 21},
    {source: 3, target: 23}, {source: 8, target: 18}
  ]
};

// Color scale for three communities
const colorScale = d3.scaleOrdinal()
  .domain([0, 1, 2])
  .range(['#1f77b4', '#ff7f0e', '#2ca02c']);

// Set up SVG
const width = document.getElementById('graph').clientWidth;
const height = 300;

const svg = d3.select('#graph')
  .append('svg')
  .attr('width', width)
  .attr('height', height)
  .attr('viewBox', `0 0 ${width * 2.5} ${height * 2.5}`)
  .attr('preserveAspectRatio', 'xMidYMid meet');

// Create force simulation
const viewBoxWidth = width * 2.5;
const viewBoxHeight = height * 2.5;
const simulation = d3.forceSimulation(data.nodes)
  .force('link', d3.forceLink(data.links).id(d => d.id).distance(50))
  .force('charge', d3.forceManyBody().strength(-300))
  .force('center', d3.forceCenter(viewBoxWidth / 2, viewBoxHeight / 2))
  .force('collision', d3.forceCollide().radius(25));

// Create links
const link = svg.append('g')
  .selectAll('line')
  .data(data.links)
  .join('line')
  .attr('class', 'link')
  .attr('stroke-width', 2);

// Create nodes
const node = svg.append('g')
  .selectAll('circle')
  .data(data.nodes)
  .join('circle')
  .attr('class', 'node')
  .attr('r', 12)
  .attr('fill', d => colorScale(d.community))
  .call(drag(simulation));

// Add labels
const label = svg.append('g')
  .selectAll('text')
  .data(data.nodes)
  .join('text')
  .attr('class', 'node-label')
  .text(d => d.id)
  .attr('dy', 4);

// Add tooltip
node.append('title')
  .text(d => `Node ${d.id}\nCommunity ${d.community}`);

// Update positions on tick
simulation.on('tick', () => {
  link
    .attr('x1', d => d.source.x)
    .attr('y1', d => d.source.y)
    .attr('x2', d => d.target.x)
    .attr('y2', d => d.target.y);

  node
    .attr('cx', d => d.x)
    .attr('cy', d => d.y);

  label
    .attr('x', d => d.x)
    .attr('y', d => d.y);
});

// Drag functionality
function drag(simulation) {
  function dragstarted(event) {
    if (!event.active) simulation.alphaTarget(0.3).restart();
    event.subject.fx = event.subject.x;
    event.subject.fy = event.subject.y;
  }

  function dragged(event) {
    event.subject.fx = event.x;
    event.subject.fy = event.y;
  }

  function dragended(event) {
    if (!event.active) simulation.alphaTarget(0);
    event.subject.fx = null;
    event.subject.fy = null;
  }

  return d3.drag()
    .on('start', dragstarted)
    .on('drag', dragged)
    .on('end', dragended);
}
</script>


## Our Approach: SBM-Based Loss Functions for GNNs

Traditional community detection methods like Louvain and spectral clustering are effective but don't leverage the representation learning capabilities of Graph Neural Networks. Meanwhile, existing GNN approaches for community detection often use heuristic loss functions that may not directly optimize for community structure quality.

### Stochastic Block Models as Loss Functions

Our key insight is that **Stochastic Block Models (SBMs)** provide a principled way to evaluate partition quality through their likelihood functions. SBMs are generative models that describe how random graphs are created based on community structure. Since SBM likelihood functions are:

1. **Well-defined**: They measure how well a partition explains the observed graph structure
2. **Differentiable**: They can be used as loss functions for gradient-based optimization
3. **Theoretically grounded**: They're based on statistical principles rather than heuristics

We can use them directly as loss functions for training GNNs in an unsupervised manner.

### Architecture-Agnostic Framework

Our approach is architecture-agnostic - it works with any GNN that outputs node embeddings. The training process:

1. GNN produces node embeddings from the input graph
2. Embeddings are mapped to soft community assignments
3. SBM likelihood evaluates the quality of these assignments
4. Gradients flow back through the network to improve embeddings

This framework allows different GNN architectures (GCN, GAT, GraphSAINT, etc.) to be trained for community detection without modifying their core structure.

### Results

Our experiments across multiple datasets show that SBM-based loss functions produce competitive results compared to existing community detection methods, while providing the benefits of:

- **End-to-end training**: No separate clustering step needed
- **Scalability**: Leverages mini-batching and GPU acceleration
- **Flexibility**: Works with various GNN architectures
- **Interpretability**: Loss directly measures partition quality via statistical likelihood

## Why This Matters

Community detection is crucial for understanding network structure in domains ranging from social networks to biological systems. By combining the representation learning power of GNNs with the theoretical foundation of SBMs, we create a framework that's both principled and practical.

The code and experiments are available in our <a href="https://openreview.net/forum?id=T1vdfm1THf" target="_blank">LoG 2025 paper</a>.
