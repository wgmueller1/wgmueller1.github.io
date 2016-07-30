---
layout: post
title: Dynamic Time Warping
description: "Temporal semantic analysis"
modified: 2014-07-19
tags: [intro, beginner, jekyll, tutorial]
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


## Dynamic Time Warping

Dyanmic Time Warping is technique which is often used in classification or clustering of time series.  DTW is used as a measure of similarity for two sequences.  In essence, the DTW algorithm stretches or compresses the sequences locally in order to make the two sequences resememble each other as closely as possible.<br>

Suppose we have two time series which consist of $$n$$ and $$m$$ observations, these are often referred to as the "query" and the "reference" series.

<center>
$$x_t=x_1,x_2,x_3, \dots x_n$$
$$y_t=y_1,y_2,y_3, \dots y_m$$
</center>

We may define a function $$f(x_i,y_j)=d(i,j) \geq 0$$ as a local dissimilarity measure.  This is the only input required for the DTW algorithm.  Typically, euclidean distance is used as the dissimilarity measure, although there are other definitions which maybe useful as well.

The key to DTW is what is known as a warping curve $$\phi(k)=(\phi_x(k),\phi_y(k))$$ which remap the time indices of $$x$$ and $$y$$.Given the warping curve, $$\phi$$, one may compute the average acculmulated distance between two time series.

<img src="/images/RPlot02.png">
<br>

<img src="/images/warp.png">

## Temporal Semantic Analysis


