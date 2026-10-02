---
title: "Embedding HTML Simulations"
summary: "Testing the html shortcode by embedding a self-contained HTML simulation into a blog post."
categories: ["Post", "Blog"]
tags: ["shortcode", "html", "embed"]
date: 2026-09-07
draft: True
---

The `html` shortcode embeds a standalone HTML file into a blog post via an iframe. This is handy for interactive simulations and demos.

## 3d simulation 

{{< html src="3d-sim.html" height="500px" title="Parameter Space Topology">}}

## Custom heiight embed

{{< html src="/simulations/demo-sim.html" height="400px" title="Smaller simulation" >}}


## Test Chart
{{< html src="sims/chart-ex.html" height="500px" title="Charting Test" >}}

## GIF Test

{{< figure src="/images/test.gif" >}}

## Prefil and decode

{{< html src="sims/prefil-decode-sim.html" height="500px" >}}


