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

{{< html-content src="3d-sim.html" height="500px" title="Parameter Space Topology">}}

## Custom heiight embed

{{< html-content src="/simulations/demo-sim.html" height="400px" title="Smaller simulation" >}}


## Test Chart
{{< html-content src="sims/chart-ex.html" height="500px" title="Charting Test" >}}

## GIF Test

{{< figure src="/images/test.gif" >}}

## Autoregressive Language model

{{< html-content src="sims/auto-regressive-llm.html" height="300px" title="LLM" >}}



## Prefil and decode

{{< html-content src="sims/prefil-decode-sim.html" height="500px" >}}


