---
title: "The KV Cache: Why a Token's Key and Value Live Forever"
summary: "Autoregressive generation, self-attention, the causal mask, and how the KV cache follows from them: once a token's key and value are computed, no later token can change them, so the model stores them and reuses them for the rest of generation. Includes the cache size formula and what it costs at long context."
categories: ["Post", "Blog"]
tags: ["llm", "inference", "kv-cache", "self-attention", "transformer"]
date: 2026-09-10
draft: true
---

{{< katex >}}

In [the prefill and decode post](/posts/prefil-and-decode/), the KV cache is the thing prefill fills and decode grows by one entry per step. This post covers where the cache comes from: why a transformer needs it, what it stores, and how much memory it takes.

The short version: generation is a loop, and a causal mask means a token's key and value vectors depend only on the tokens up to and including that one. Later tokens can never change them. So the model computes each key and value once, stores them, and reads them back on every later step. The size formula and the memory cost of long context both follow from that.

## Generation is a loop

A language model produces text one token at a time. Each step takes the whole sequence so far, predicts the next token, appends it, and runs again. The model's own output becomes part of its next input. This is called autoregressive generation.

{{< simulation title="Autoregressive generation: each step reads the full sequence and adds one token" height="300px" >}}
<svg viewBox="0 0 760 286" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="a1" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="286" rx="10" fill="#f8fafc"/>
<text x="240" y="26" font-size="12" font-weight="700" fill="#475569" text-anchor="middle">sequence so far (input)</text>
<text x="510" y="26" font-size="12" font-weight="700" fill="#475569" text-anchor="middle">model</text>
<text x="652" y="26" font-size="12" font-weight="700" fill="#475569" text-anchor="middle">next token</text>
<text x="20" y="67" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">step 1</text>
<rect x="80" y="44" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="108.0" y="66.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="144" y="44" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="172.0" y="66.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">cat</text>
<rect x="208" y="44" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="236.0" y="66.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">sat</text>
<line x1="404" y1="62" x2="448" y2="62" stroke="#475569" stroke-width="1.5" marker-end="url(#a1)"/>
<rect x="450" y="44" width="120" height="36" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="510.0" y="66.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="570" y1="62" x2="618" y2="62" stroke="#475569" stroke-width="1.5" marker-end="url(#a1)"/>
<rect x="620" y="44" width="64" height="36" rx="6" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/><text x="652.0" y="66.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">on</text>
<text x="20" y="139" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">step 2</text>
<rect x="80" y="116" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="108.0" y="138.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="144" y="116" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="172.0" y="138.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">cat</text>
<rect x="208" y="116" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="236.0" y="138.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">sat</text>
<rect x="272" y="116" width="56" height="36" rx="6" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/><text x="300.0" y="138.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">on</text>
<line x1="404" y1="134" x2="448" y2="134" stroke="#475569" stroke-width="1.5" marker-end="url(#a1)"/>
<rect x="450" y="116" width="120" height="36" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="510.0" y="138.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="570" y1="134" x2="618" y2="134" stroke="#475569" stroke-width="1.5" marker-end="url(#a1)"/>
<rect x="620" y="116" width="64" height="36" rx="6" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/><text x="652.0" y="138.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">the</text>
<text x="20" y="211" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">step 3</text>
<rect x="80" y="188" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="108.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="144" y="188" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="172.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">cat</text>
<rect x="208" y="188" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="236.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">sat</text>
<rect x="272" y="188" width="56" height="36" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="300.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">on</text>
<rect x="336" y="188" width="56" height="36" rx="6" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/><text x="364.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">the</text>
<line x1="404" y1="206" x2="448" y2="206" stroke="#475569" stroke-width="1.5" marker-end="url(#a1)"/>
<rect x="450" y="188" width="120" height="36" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="510.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="570" y1="206" x2="618" y2="206" stroke="#475569" stroke-width="1.5" marker-end="url(#a1)"/>
<rect x="620" y="188" width="64" height="36" rx="6" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/><text x="652.0" y="210.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">mat</text>
<text x="380" y="268" font-size="12" font-weight="400" fill="#64748b" text-anchor="middle">Purple: the token the model just predicted. At the next step it is appended to the input.</text>
</svg>
{{< /simulation >}}

Compare step 1 with step 2. The input at step 2 is the input from step 1 plus one new token, and step 3 does the same again. If the model reprocessed the full sequence from scratch each time, generating n tokens would cost 1 + 2 + … + n token forward passes, which grows as n². Almost all of that work is repeated. The KV cache removes the repetition, and to see how, we need to look inside a layer.

## Self-attention in one sentence

Attention is the only operation in a transformer layer where one token reads another (the feed-forward part handles each token on its own). Each token looks at the earlier tokens, decides how much weight to give each one, and takes a weighted mix of them as its new representation.

## Three projections per token

To do that lookup, each token's hidden state is multiplied by three learned matrices, giving three vectors:

{{< simulation title="Three learned projections per token: query, key, value" height="205px" >}}
<svg viewBox="0 0 760 190" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="a3" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="190" rx="10" fill="#f8fafc"/>
<rect x="20" y="70" width="110" height="50" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/><text x="75.0" y="99.9" font-size="14" font-weight="700" fill="#1f2937" text-anchor="middle">xᵢ</text>
<text x="75" y="138" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">hidden state of token i</text>
<line x1="130" y1="95" x2="188" y2="38" stroke="#475569" stroke-width="1.5" marker-end="url(#a3)"/>
<rect x="190" y="20" width="90" height="36" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="235.0" y="42.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">W_Q</text>
<line x1="280" y1="38" x2="348" y2="38" stroke="#b91c1c" stroke-width="2" marker-end="url(#a3)"/>
<rect x="350" y="20" width="130" height="36" rx="6" fill="#fee2e2" stroke="#b91c1c" stroke-width="2"/><text x="415.0" y="42.5" font-size="13" font-weight="700" fill="#7f1d1d" text-anchor="middle">qᵢ  query</text>
<text x="500" y="42" font-size="12" font-weight="400" fill="#475569" text-anchor="start">what am I looking for?</text>
<line x1="130" y1="95" x2="188" y2="95" stroke="#475569" stroke-width="1.5" marker-end="url(#a3)"/>
<rect x="190" y="77" width="90" height="36" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="235.0" y="99.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">W_K</text>
<line x1="280" y1="95" x2="348" y2="95" stroke="#15803d" stroke-width="2" marker-end="url(#a3)"/>
<rect x="350" y="77" width="130" height="36" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="2"/><text x="415.0" y="99.5" font-size="13" font-weight="700" fill="#14532d" text-anchor="middle">kᵢ  key</text>
<text x="500" y="99" font-size="12" font-weight="400" fill="#475569" text-anchor="start">what do I advertise?</text>
<line x1="130" y1="95" x2="188" y2="152" stroke="#475569" stroke-width="1.5" marker-end="url(#a3)"/>
<rect x="190" y="134" width="90" height="36" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="235.0" y="156.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">W_V</text>
<line x1="280" y1="152" x2="348" y2="152" stroke="#b45309" stroke-width="2" marker-end="url(#a3)"/>
<rect x="350" y="134" width="130" height="36" rx="6" fill="#fef3c7" stroke="#b45309" stroke-width="2"/><text x="415.0" y="156.6" font-size="13" font-weight="700" fill="#92400e" text-anchor="middle">vᵢ  value</text>
<text x="500" y="156" font-size="12" font-weight="400" fill="#475569" text-anchor="start">what do I hand over if picked?</text>
</svg>
{{< /simulation >}}

$$
\begin{aligned}
\mathbf{q}_i &= \mathbf{x}_i \mathbf{W}_Q \\
\mathbf{k}_i &= \mathbf{x}_i \mathbf{W}_K \\
\mathbf{v}_i &= \mathbf{x}_i \mathbf{W}_V
\end{aligned}
$$

The query is what this token is looking for. The key is what it advertises to other tokens. The value is what it hands over when another token picks it. Attention is a soft lookup: the query is matched against keys, the matches become weights, and the weights mix the values. Training decides what each projection learns to encode.

## How attention computes its output

Token i produces its output in three steps:

1. Score: take the dot product of its query with the key of every token it can see, itself included.
2. Normalize: run a softmax over those scores so the weights are positive and sum to 1.
3. Mix: add up the value vectors, each multiplied by its weight.

$$
\begin{aligned}
\text{score}(i, j) &= \frac{\mathbf{q}_i \cdot \mathbf{k}_j}{\sqrt{d_k}} \\
\text{weights}(i, j) &= \text{softmax}_j\big(\text{score}(i, \cdot)\big) \\
\mathbf{out}_i &= \sum_{j \le i} \text{weights}(i, j)\, \mathbf{v}_j
\end{aligned}
$$

The √d_k keeps dot products from growing with the head dimension, which would push the softmax toward putting all its weight on one token.

Here is token 4 ("on") doing this over the sequence "The cat sat on":

{{< simulation title="One token attends to every token up to and including itself" height="320px" >}}
<svg viewBox="0 0 760 304" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="a4" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="304" rx="10" fill="#f8fafc"/>
<text x="55" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">query</text>
<text x="198" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">keys</text>
<text x="350" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">attention weights</text>
<text x="528" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">values</text>
<text x="704" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">output</text>
<rect x="20" y="135.0" width="70" height="40" rx="6" fill="#fee2e2" stroke="#b91c1c" stroke-width="2"/><text x="55.0" y="159.9" font-size="14" font-weight="700" fill="#7f1d1d" text-anchor="middle">q₄</text>
<text x="55" y="193.0" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">token “on”</text>
<line x1="90" y1="155.0" x2="148" y2="62" stroke="#b91c1c" stroke-width="1.2" marker-end="url(#a4)"/>
<rect x="150" y="44" width="96" height="36" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="2"/><text x="198.0" y="66.5" font-size="13" font-weight="700" fill="#14532d" text-anchor="middle">k₁ · The</text>
<line x1="246" y1="62" x2="288" y2="62" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="290" y="52" width="120" height="20" rx="4" fill="#e2e8f0"/>
<rect x="290" y="52" width="6" height="20" rx="4" fill="#15803d"/>
<text x="418" y="67" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">0.05</text>
<line x1="456" y1="62" x2="478" y2="62" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="480" y="44" width="96" height="36" rx="6" fill="#fef3c7" stroke="#b45309" stroke-width="2"/><text x="528.0" y="66.5" font-size="13" font-weight="700" fill="#92400e" text-anchor="middle">v₁ · The</text>
<line x1="576" y1="62" x2="612" y2="155.0" stroke="#b45309" stroke-width="1.2"/>
<line x1="90" y1="155.0" x2="148" y2="124" stroke="#b91c1c" stroke-width="1.4" marker-end="url(#a4)"/>
<rect x="150" y="106" width="96" height="36" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="2"/><text x="198.0" y="128.6" font-size="13" font-weight="700" fill="#14532d" text-anchor="middle">k₂ · cat</text>
<line x1="246" y1="124" x2="288" y2="124" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="290" y="114" width="120" height="20" rx="4" fill="#e2e8f0"/>
<rect x="290" y="114" width="12" height="20" rx="4" fill="#15803d"/>
<text x="418" y="129" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">0.10</text>
<line x1="456" y1="124" x2="478" y2="124" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="480" y="106" width="96" height="36" rx="6" fill="#fef3c7" stroke="#b45309" stroke-width="2"/><text x="528.0" y="128.6" font-size="13" font-weight="700" fill="#92400e" text-anchor="middle">v₂ · cat</text>
<line x1="576" y1="124" x2="612" y2="155.0" stroke="#b45309" stroke-width="1.4"/>
<line x1="90" y1="155.0" x2="148" y2="186" stroke="#b91c1c" stroke-width="3.4" marker-end="url(#a4)"/>
<rect x="150" y="168" width="96" height="36" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="2"/><text x="198.0" y="190.6" font-size="13" font-weight="700" fill="#14532d" text-anchor="middle">k₃ · sat</text>
<line x1="246" y1="186" x2="288" y2="186" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="290" y="176" width="120" height="20" rx="4" fill="#e2e8f0"/>
<rect x="290" y="176" width="72" height="20" rx="4" fill="#15803d"/>
<text x="418" y="191" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">0.60</text>
<line x1="456" y1="186" x2="478" y2="186" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="480" y="168" width="96" height="36" rx="6" fill="#fef3c7" stroke="#b45309" stroke-width="2"/><text x="528.0" y="190.6" font-size="13" font-weight="700" fill="#92400e" text-anchor="middle">v₃ · sat</text>
<line x1="576" y1="186" x2="612" y2="155.0" stroke="#b45309" stroke-width="3.4"/>
<line x1="90" y1="155.0" x2="148" y2="248" stroke="#b91c1c" stroke-width="2.0" marker-end="url(#a4)"/>
<rect x="150" y="230" width="96" height="36" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="2"/><text x="198.0" y="252.6" font-size="13" font-weight="700" fill="#14532d" text-anchor="middle">k₄ · on</text>
<line x1="246" y1="248" x2="288" y2="248" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="290" y="238" width="120" height="20" rx="4" fill="#e2e8f0"/>
<rect x="290" y="238" width="30" height="20" rx="4" fill="#15803d"/>
<text x="418" y="253" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">0.25</text>
<line x1="456" y1="248" x2="478" y2="248" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="480" y="230" width="96" height="36" rx="6" fill="#fef3c7" stroke="#b45309" stroke-width="2"/><text x="528.0" y="252.6" font-size="13" font-weight="700" fill="#92400e" text-anchor="middle">v₄ · on</text>
<line x1="576" y1="248" x2="612" y2="155.0" stroke="#b45309" stroke-width="2.0"/>
<circle cx="630" cy="155.0" r="18" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/>
<text x="630" y="161.0" font-size="16" font-weight="700" fill="#4c1d95" text-anchor="middle">Σ</text>
<line x1="648" y1="155.0" x2="676" y2="155.0" stroke="#475569" stroke-width="1.5" marker-end="url(#a4)"/>
<rect x="678" y="135.0" width="72" height="40" rx="6" fill="#ede9fe" stroke="#6d28d9" stroke-width="2"/><text x="714.0" y="159.6" font-size="13" font-weight="700" fill="#4c1d95" text-anchor="middle">out₄</text>
<text x="380" y="290" font-size="12" font-weight="400" fill="#64748b" text-anchor="middle">weights = softmax(q₄ · kⱼ / √d). They sum to 1, and token 4 attends to itself too (j ≤ i).</text>
</svg>
{{< /simulation >}}

Token 4 puts most of its weight on "sat" and some on itself, so its output is mostly the value of "sat". The keys and values of tokens 1 to 3 were computed on earlier steps. Token 4 computes its own query, key, and value on this step, and uses the query only here.

## The causal mask

At position i, the model may attend only to positions j ≤ i. A causal mask enforces this: before the softmax, every score with j > i is set to −∞, which gives that position exactly zero weight. In practice the mask is an additive bias on the attention scores, applied in every layer.

{{< simulation title="Causal mask: a token can only attend to itself and earlier tokens" height="410px" >}}
<svg viewBox="0 0 760 396" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="a2" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="396" rx="10" fill="#f8fafc"/>
<text x="325.0" y="26" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">key position j</text>
<text x="70" y="215.0" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">query position i</text>
<text x="217.0" y="66" font-size="13" font-weight="700" fill="#15803d" text-anchor="middle">k₁</text>
<text x="271.0" y="66" font-size="13" font-weight="700" fill="#15803d" text-anchor="middle">k₂</text>
<text x="325.0" y="66" font-size="13" font-weight="700" fill="#15803d" text-anchor="middle">k₃</text>
<text x="379.0" y="66" font-size="13" font-weight="700" fill="#15803d" text-anchor="middle">k₄</text>
<text x="433.0" y="66" font-size="13" font-weight="700" fill="#15803d" text-anchor="middle">k₅</text>
<text x="178" y="108.0" font-size="13" font-weight="700" fill="#b91c1c" text-anchor="end">q₁</text>
<rect x="192" y="78" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="217.0" y="106.8" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="246" y="78" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="271.0" y="107.5" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<rect x="300" y="78" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="325.0" y="107.5" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<rect x="354" y="78" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="379.0" y="107.5" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<rect x="408" y="78" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="433.0" y="107.5" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<text x="178" y="162.0" font-size="13" font-weight="700" fill="#b91c1c" text-anchor="end">q₂</text>
<rect x="192" y="132" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="217.0" y="160.8" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="246" y="132" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="271.0" y="160.8" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="300" y="132" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="325.0" y="161.6" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<rect x="354" y="132" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="379.0" y="161.6" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<rect x="408" y="132" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="433.0" y="161.6" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<text x="178" y="216.0" font-size="13" font-weight="700" fill="#b91c1c" text-anchor="end">q₃</text>
<rect x="192" y="186" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="217.0" y="214.8" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="246" y="186" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="271.0" y="214.8" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="300" y="186" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="325.0" y="214.8" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="354" y="186" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="379.0" y="215.6" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<rect x="408" y="186" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="433.0" y="215.6" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<text x="178" y="270.0" font-size="13" font-weight="700" fill="#b91c1c" text-anchor="end">q₄</text>
<rect x="192" y="240" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="217.0" y="268.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="246" y="240" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="271.0" y="268.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="300" y="240" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="325.0" y="268.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="354" y="240" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="379.0" y="268.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="408" y="240" width="50" height="50" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="433.0" y="269.6" font-size="13" font-weight="600" fill="#64748b" text-anchor="middle">−∞</text>
<text x="178" y="324.0" font-size="13" font-weight="700" fill="#b91c1c" text-anchor="end">q₅</text>
<rect x="192" y="294" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="217.0" y="322.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="246" y="294" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="271.0" y="322.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="300" y="294" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="325.0" y="322.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="354" y="294" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="379.0" y="322.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="408" y="294" width="50" height="50" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="433.0" y="322.9" font-size="11" font-weight="600" fill="#14532d" text-anchor="middle">score</text>
<rect x="188" y="238" width="274" height="54" rx="8" fill="none" stroke="#b91c1c" stroke-width="3"/>
<text x="490" y="220" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">Row q₄ (token 4)</text>
<text x="490" y="237" font-size="12" font-weight="400" fill="#475569" text-anchor="start">can read k₁ to k₄.</text>
<text x="490" y="254" font-size="12" font-weight="400" fill="#475569" text-anchor="start">k₅ is a future token, so its</text>
<text x="490" y="271" font-size="12" font-weight="400" fill="#475569" text-anchor="start">score becomes −∞ before the</text>
<text x="490" y="288" font-size="12" font-weight="400" fill="#475569" text-anchor="start">softmax and its weight is 0.</text>
<rect x="190" y="366" width="16" height="16" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="198.0" y="378.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle"></text>
<text x="212" y="378" font-size="12" font-weight="400" fill="#475569" text-anchor="start">allowed</text>
<rect x="290" y="366" width="16" height="16" rx="6" fill="#e2e8f0" stroke="#94a3b8" stroke-width="1.5"/><text x="298.0" y="378.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle"></text>
<text x="312" y="378" font-size="12" font-weight="400" fill="#475569" text-anchor="start">masked</text>
</svg>
{{< /simulation >}}

You need the mask even though a generating model has no future tokens to look at. Training processes whole sequences in parallel, and without the mask, position i could read the very token it is supposed to predict. The mask also makes the model a valid probability distribution over sequences, because the autoregressive factorization P(x₁) · P(x₂ | x₁) · P(x₃ | x₁, x₂) … only holds if every position uses earlier positions alone. So the mask is part of the architecture, and it applies when the model processes a prompt in prefill too.

It also has a consequence the cache depends on: nothing a token computes is ever influenced by the tokens after it.

## Multi-head attention

Everything above describes one attention head. A real layer runs several heads in parallel, each with its own W_Q, W_K, and W_V. The model dimension is split into h smaller subspaces, each head attends independently, and the outputs are concatenated at the end.

Different heads learn to look for different things. One might track subject-verb agreement, another which noun a pronoun refers to, another the previous few tokens. Nobody assigns these roles; they emerge in training.

For caching, this means the model stores one K and one V per token per head, in every layer. A model with 32 heads and 24 layers keeps 768 K vectors and 768 V vectors for each token. The cache size formula below scales with both numbers.

## Where the cache comes from

Now the observation the post is built on. Start with what each vector depends on.

In the first layer, x_i is the token's embedding (plus its position information), so q_i, k_i, and v_i depend only on that token and its position. In the second layer, x_i is the first layer's output at position i, which has already mixed in tokens 1 to i through attention. So from layer 2 on, k_i and v_i depend on every token up to position i.

Because of the causal mask, they never depend on anything after position i. When token i + 1 arrives, no number computed for tokens 1 to i changes. Their keys and values, in every layer and every head, are fixed at the moment they are computed. Recomputing them at a later step would give identical results.

The query behaves differently. Token i's query is used once, at step i, to read the keys and values of the tokens before it. Nothing later needs it. So queries are always fresh, and keys and values stay valid for the rest of the sequence.

The cache stores each token's keys and values when they are computed and reads them back on every later step. Here is the same five-step generation, without and with a cache:

{{< simulation title="Keys and values for old tokens: recomputed every step vs read from the cache" height="370px" >}}
<svg viewBox="0 0 760 354" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="a5" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="354" rx="10" fill="#f8fafc"/>
<text x="202.0" y="26" font-size="14" font-weight="700" fill="#1f2937" text-anchor="middle">Without a cache</text>
<text x="86" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 1</text>
<text x="144" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 2</text>
<text x="202" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 3</text>
<text x="260" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 4</text>
<text x="318" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 5</text>
<text x="52" y="90" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 1</text>
<rect x="60" y="72" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="86.0" y="91.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="52" y="128" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 2</text>
<rect x="60" y="110" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="86.0" y="129.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="118" y="110" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="144.0" y="129.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="52" y="166" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 3</text>
<rect x="60" y="148" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="86.0" y="167.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="118" y="148" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="144.0" y="167.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="176" y="148" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="202.0" y="167.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="52" y="204" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 4</text>
<rect x="60" y="186" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="86.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="118" y="186" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="144.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="176" y="186" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="202.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="234" y="186" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="260.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="52" y="242" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 5</text>
<rect x="60" y="224" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="86.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="118" y="224" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="144.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="176" y="224" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="202.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="234" y="224" width="52" height="32" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="260.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">redo</text>
<rect x="292" y="224" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="318.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="202.0" y="284" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">15 K/V computations</text>
<text x="582.0" y="26" font-size="14" font-weight="700" fill="#1f2937" text-anchor="middle">With a KV cache</text>
<text x="466" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 1</text>
<text x="524" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 2</text>
<text x="582" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 3</text>
<text x="640" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 4</text>
<text x="698" y="62" font-size="11" font-weight="400" fill="#64748b" text-anchor="middle">tok 5</text>
<text x="432" y="90" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 1</text>
<rect x="440" y="72" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="466.0" y="91.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="432" y="128" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 2</text>
<rect x="440" y="110" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="466.0" y="129.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="498" y="110" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="524.0" y="129.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="432" y="166" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 3</text>
<rect x="440" y="148" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="466.0" y="167.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="498" y="148" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="524.0" y="167.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="556" y="148" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="582.0" y="167.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="432" y="204" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 4</text>
<rect x="440" y="186" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="466.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="498" y="186" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="524.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="556" y="186" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="582.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="614" y="186" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="640.0" y="205.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="432" y="242" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">step 5</text>
<rect x="440" y="224" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="466.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="498" y="224" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="524.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="556" y="224" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="582.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="614" y="224" width="52" height="32" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="640.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">cached</text>
<rect x="672" y="224" width="52" height="32" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="698.0" y="243.8" font-size="11" font-weight="600" fill="#1f2937" text-anchor="middle">new</text>
<text x="582.0" y="284" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">5 K/V computations</text>
<rect x="60" y="318" width="16" height="16" rx="6" fill="#dbeafe" stroke="#1d4ed8" stroke-width="1.5"/><text x="68.0" y="330.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle"></text>
<text x="82" y="330" font-size="12" font-weight="400" fill="#475569" text-anchor="start">new: computed this step</text>
<rect x="290" y="318" width="16" height="16" rx="6" fill="#fed7aa" stroke="#c2410c" stroke-width="1.5"/><text x="298.0" y="330.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle"></text>
<text x="312" y="330" font-size="12" font-weight="400" fill="#475569" text-anchor="start">redo: recomputed from scratch</text>
<rect x="540" y="318" width="16" height="16" rx="6" fill="#dcfce7" stroke="#15803d" stroke-width="1.5"/><text x="548.0" y="330.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle"></text>
<text x="562" y="330" font-size="12" font-weight="400" fill="#475569" text-anchor="start">cached: read, not recomputed</text>
</svg>
{{< /simulation >}}

Without a cache, step 5 pushes all five tokens through the whole network again to rebuild keys and values that step 4 already had. With a cache, step 5 processes only the new token, projects its query, key, and value, appends the key and value to the cache, and reads everything else from it. The saving is not just the projections: the old tokens skip every layer.

You can see the asymmetry in the tensor shapes during decode. Q has one position, because only the new token needs a query. K and V have one entry per token in the context.

| Tensor | Shape during a decode step |
| --- | --- |
| Q | (1, heads, head_dim) |
| K | (seq_len, heads, head_dim), grows by one each step |
| V | (seq_len, heads, head_dim), grows by one each step |

## How big the cache is

For one request, take a model with L layers, H key/value heads per layer, head dimension d, and a current sequence length S. The cache holds:

$$
\text{cache size} = 2 \cdot L \cdot H \cdot d \cdot S \cdot \text{bytes per value}
$$

The 2 is for K and V. Bytes per value is 2 for fp16 or bf16, 4 for fp32, and 1 for int8. S counts everything in context: the prompt plus everything generated so far.

Take Llama 3 70B: 80 layers, head dimension 128, bf16. It has 64 query heads but only 8 key/value heads, because it uses grouped-query attention (more on that below). Per token:

$$
2 \cdot 80 \cdot 8 \cdot 128 \cdot 2 \text{ bytes} = 327{,}680 \text{ bytes} \approx 0.33 \text{ MB}
$$

For comparison, here is the same model if every one of its 64 heads kept its own K and V:

| Key/value heads | Cache per token | Cache for a 100k-token request |
| --- | --- | --- |
| 64 (one per query head) | ≈ 2.6 MB | ≈ 262 GB |
| 8 (what Llama 3 70B uses) | ≈ 0.33 MB | ≈ 33 GB |

The model weights take about 140 GB in bf16. So a single 100k-token request adds 33 GB on top of that, and a 128k-token request adds about 43 GB. Without grouped-query attention, one 100k-token request would need nearly twice the memory of the weights.

This is why long context is expensive. Each decode step has to read the entire cache from GPU memory, so step time rises with context length. The cache also competes with the weights and with other requests for the same memory, so every extra token of context lowers the number of requests a GPU can serve at once.

The cache grows linearly with sequence length, and the per-token cost comes entirely from the model's architecture and precision. You can compute it from a config file without running a benchmark.

## Variations on the cache

The formula and the argument above stay the same in all of these. They change how the cache is stored, shared, or shrunk.

- **Grouped-query attention (GQA)**: several query heads share one K head and one V head. Llama 3 70B has 64 query heads and 8 K/V heads, so its cache is 8× smaller than the naive formula predicts, with little quality loss. Most current models do this.
- **Multi-query attention (MQA)**: the extreme version, with a single K head and a single V head shared by all query heads. The cache is smaller still, at a somewhat larger quality cost.
- **Paged attention** (vLLM and similar servers): the cache is split into fixed-size blocks, like virtual memory pages, instead of one contiguous tensor per request. Blocks are allocated and freed as needed, which avoids fragmentation when many requests are in flight.
- **Sliding window attention** (the original Mistral 7B): each token attends only to the last N tokens, so older keys and values are evicted as new ones arrive. Memory stays bounded, and long-range context is lost.
- **Prompt caching**: when two requests share a long prefix, such as a system prompt, the second can start with the first one's cached keys and values for that prefix and skip most of its prefill.
- **Eviction**: when a conversation outgrows memory, the simplest policy drops the oldest entries. The model loses access to that context. Some newer policies try to keep the entries that get the most attention instead of the oldest ones.

## Estimating the cache for your model

To estimate the cache cost for a model you plan to serve, open its `config.json` and read `num_hidden_layers`, `num_key_value_heads`, and the head dimension (`hidden_size` divided by `num_attention_heads`, unless the config sets `head_dim`). Plug them into the formula with your serving precision. For how the cache is filled and consumed at serving time, see [the prefill and decode post](/posts/prefil-and-decode/).