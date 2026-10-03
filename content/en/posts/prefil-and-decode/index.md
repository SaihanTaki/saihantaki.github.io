---
title: "Prefill and Decode: How an LLM Serves a Single Request"
summary: "An LLM request runs in two phases: prefill reads the whole prompt in one parallel pass, and decode writes the reply one token at a time. This post walks through both, why they hit different hardware limits, and how that shapes TTFT, tokens per second, and serving design."
categories: ["Post", "Blog"]
tags: ["llm", "inference", "prefill", "decode", "kv-cache", "autoregressive"]
date: 2026-10-03
draft: false
---

{{< katex >}}

Send `The quick brown fox` to an LLM and two different things happen. First the model reads your prompt. Then it writes the reply one token at a time. These are the **prefill** and **decode** phases. They have different costs and hit different hardware limits, and that explains why long prompts are slow to start, why streaming speed barely depends on prompt length, and why serving systems spend so much effort on KV cache memory.

I'll use one example throughout. The prompt is **The quick brown fox** (four tokens) and the model's reply is **jumps over the lazy dog** (five tokens). Where the KV cache comes from, needs a post of its own and is out of scope here. This post only uses the cache, and I explain what it does as we go.

## Generation is a loop

A language model writes one token at a time. At each step it reads all the text so far, predicts the next token, appends it, and repeats. This is called autoregressive generation: the model's own output becomes part of its next input.

{{< simulation title="Autoregressive Generation">}}
<svg viewBox="0 0 760 356" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="m1" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="356" rx="10" fill="none"/>
<text x="300" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">context fed to the model</text>
<text x="585" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">model</text>
<text x="704" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">predicted</text>
<text x="14" y="66" font-size="12" font-weight="700" fill="#4338ca" text-anchor="start">prefill</text>
<rect x="84" y="44" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="108.0" y="65.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="136" y="44" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="160.0" y="65.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<rect x="188" y="44" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="212.0" y="65.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<rect x="240" y="44" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="264.0" y="65.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<line x1="500" y1="61" x2="528" y2="61" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="530" y="44" width="110" height="34" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="585.0" y="65.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="640" y1="61" x2="664" y2="61" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="666" y="44" width="76" height="34" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="704.0" y="65.5" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">jumps</text>
<text x="14" y="122" font-size="12" font-weight="700" fill="#047857" text-anchor="start">decode 1</text>
<rect x="84" y="100" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="108.0" y="121.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="136" y="100" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="160.0" y="121.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<rect x="188" y="100" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="212.0" y="121.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<rect x="240" y="100" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="264.0" y="121.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<rect x="292" y="100" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="316.0" y="121.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">jumps</text>
<line x1="500" y1="117" x2="528" y2="117" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="530" y="100" width="110" height="34" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="585.0" y="121.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="640" y1="117" x2="664" y2="117" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="666" y="100" width="76" height="34" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="704.0" y="121.5" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">over</text>
<text x="14" y="178" font-size="12" font-weight="700" fill="#047857" text-anchor="start">decode 2</text>
<rect x="84" y="156" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="108.0" y="177.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="136" y="156" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="160.0" y="177.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<rect x="188" y="156" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="212.0" y="177.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<rect x="240" y="156" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="264.0" y="177.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<rect x="292" y="156" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="316.0" y="177.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">jumps</text>
<rect x="344" y="156" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="368.0" y="177.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">over</text>
<line x1="500" y1="173" x2="528" y2="173" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="530" y="156" width="110" height="34" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="585.0" y="177.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="640" y1="173" x2="664" y2="173" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="666" y="156" width="76" height="34" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="704.0" y="177.6" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">the</text>
<text x="14" y="234" font-size="12" font-weight="700" fill="#047857" text-anchor="start">decode 3</text>
<rect x="84" y="212" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="108.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="136" y="212" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="160.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<rect x="188" y="212" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="212.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<rect x="240" y="212" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="264.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<rect x="292" y="212" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="316.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">jumps</text>
<rect x="344" y="212" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="368.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">over</text>
<rect x="396" y="212" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="420.0" y="233.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">the</text>
<line x1="500" y1="229" x2="528" y2="229" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="530" y="212" width="110" height="34" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="585.0" y="233.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="640" y1="229" x2="664" y2="229" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="666" y="212" width="76" height="34" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="704.0" y="233.6" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">lazy</text>
<text x="14" y="290" font-size="12" font-weight="700" fill="#047857" text-anchor="start">decode 4</text>
<rect x="84" y="268" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="108.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="136" y="268" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="160.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<rect x="188" y="268" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="212.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<rect x="240" y="268" width="48" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="264.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<rect x="292" y="268" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="316.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">jumps</text>
<rect x="344" y="268" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="368.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">over</text>
<rect x="396" y="268" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="420.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">the</text>
<rect x="448" y="268" width="48" height="34" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/><text x="472.0" y="289.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">lazy</text>
<line x1="500" y1="285" x2="528" y2="285" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="530" y="268" width="110" height="34" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="585.0" y="289.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<line x1="640" y1="285" x2="664" y2="285" stroke="#475569" stroke-width="1.5" marker-end="url(#m1)"/>
<rect x="666" y="268" width="76" height="34" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="704.0" y="289.6" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">dog</text>
<text x="380" y="338" font-size="12" font-weight="400" fill="#64748b" text-anchor="middle">Blue: prompt. Green: tokens the model wrote earlier. Amber: the token it predicts from this context.</text>
</svg>
{{< /simulation >}}

The first row is the prompt going in. The model predicts `jumps`. In the second row, `jumps` has joined the context and the model predicts `over`, and so on. Each row's input is the previous row's input plus one token.

In probability terms, the reply is written as a chain, and each link conditions on everything before it:

$$
P(\text{jumps, over, the, lazy, dog} \mid \text{prompt}) = P(\text{jumps} \mid \text{prompt}) \cdot P(\text{over} \mid \text{prompt, jumps}) \cdot P(\text{the} \mid \text{prompt, jumps, over}) \cdots
$$

Each factor is a full probability distribution over the model's vocabulary. After `The quick brown`, the model puts most of its probability on words that fit the idiom. A token depends only on the tokens before it, never on tokens after it. That one-directional rule is what makes the rest of this post work.

Taken literally, the diagram describes a wasteful process. Row 5 feeds eight tokens through the model, and seven of them were already processed in row 4. The fix is to remember the intermediate results of the old tokens in a **KV cache**, which stores two vectors per token per layer, a key and a value, that later tokens read when they attend. With a cache, the model has two distinct jobs:

- **Prefill**: process the whole prompt once, fill the cache, and predict the first token. That is row 1.
- **Decode**: for every later token, push only the newest token through the model, reuse the cache for everything older, and add one new entry. That is rows 2 to 5.

## Prefill: the whole prompt at once

The prompt tokens are all known up front, so nothing forces the model to read them one by one. Prefill runs them through the network together.

Suppose the model has 4 layers. Within a layer, all 4 tokens are processed side by side. The layers themselves still run in order, because layer 2 needs the output of layer 1.

{{< simulation title="Prefill: tokens run in parallel inside each layer, layers run in sequence">}}
<svg viewBox="0 0 760 402" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="m2" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="402" rx="10" fill="none"/>
<text x="343" y="24" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">4 prompt tokens, side by side</text>
<text x="112" y="69" font-size="13" font-weight="700" fill="#1f2937" text-anchor="end">Layer 1</text>
<rect x="130" y="44" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="178.0" y="68.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<line x1="178" y1="86" x2="178" y2="104" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="240" y="44" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="288.0" y="68.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<line x1="288" y1="86" x2="288" y2="104" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="350" y="44" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="398.0" y="68.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<line x1="398" y1="86" x2="398" y2="104" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="460" y="44" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="508.0" y="68.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<line x1="508" y1="86" x2="508" y2="104" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<text x="580" y="69" font-size="13" font-weight="700" fill="#4338ca" text-anchor="start">round 1</text>
<text x="112" y="131" font-size="13" font-weight="700" fill="#1f2937" text-anchor="end">Layer 2</text>
<rect x="130" y="106" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="178.0" y="130.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<line x1="178" y1="148" x2="178" y2="166" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="240" y="106" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="288.0" y="130.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<line x1="288" y1="148" x2="288" y2="166" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="350" y="106" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="398.0" y="130.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<line x1="398" y1="148" x2="398" y2="166" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="460" y="106" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="508.0" y="130.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<line x1="508" y1="148" x2="508" y2="166" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<text x="580" y="131" font-size="13" font-weight="700" fill="#4338ca" text-anchor="start">round 2</text>
<text x="112" y="193" font-size="13" font-weight="700" fill="#1f2937" text-anchor="end">Layer 3</text>
<rect x="130" y="168" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="178.0" y="192.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<line x1="178" y1="210" x2="178" y2="228" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="240" y="168" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="288.0" y="192.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<line x1="288" y1="210" x2="288" y2="228" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="350" y="168" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="398.0" y="192.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<line x1="398" y1="210" x2="398" y2="228" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="460" y="168" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="508.0" y="192.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<line x1="508" y1="210" x2="508" y2="228" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<text x="580" y="193" font-size="13" font-weight="700" fill="#4338ca" text-anchor="start">round 3</text>
<text x="112" y="255" font-size="13" font-weight="700" fill="#1f2937" text-anchor="end">Layer 4</text>
<rect x="130" y="230" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="178.0" y="254.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<rect x="240" y="230" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="288.0" y="254.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<rect x="350" y="230" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="398.0" y="254.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<rect x="460" y="230" width="96" height="40" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="508.0" y="254.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<text x="580" y="255" font-size="13" font-weight="700" fill="#4338ca" text-anchor="start">round 4</text>
<line x1="240" y1="274" x2="240" y2="316" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<line x1="508" y1="274" x2="508" y2="316" stroke="#475569" stroke-width="1.5" marker-end="url(#m2)"/>
<rect x="140" y="318" width="200" height="40" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="240.0" y="342.6" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">KV cache: 4 entries</text>
<rect x="408" y="318" width="200" height="40" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="1.5"/><text x="508.0" y="342.6" font-size="13" font-weight="700" fill="#1f2937" text-anchor="middle">first token: “jumps”</text>
<text x="380" y="384" font-size="12" font-weight="400" fill="#64748b" text-anchor="middle">Tokens run in parallel within a layer. Layers still run in order, so there are 4 rounds, not 16.</text>
</svg>
{{< /simulation >}}

Two things come out of prefill. One is the KV cache: the keys and values of every prompt token at every layer, ready for decode to read. The other is a prediction for the first output token, taken from the last prompt position.

Prefill time grows with prompt length, because it is a large amount of arithmetic. Very short prompts don't fill the GPU, so a 4-token and a 40-token prompt take about the same time. Once the prompt is big enough to keep the GPU busy, time grows roughly in proportion to its length. A 10,000-token prompt makes you wait noticeably longer than a 100-token one, and the wait happens before any output appears.

## Decode: one token per step

After prefill, the model predicts one token at a time, and each token depends on the one before it, so these steps cannot run in parallel. At each step only the newest token goes through the model. It attends to the cache for everything older, and its own key and value are added to the cache.

Step through the example below. The first step is prefill, and the next four are decode steps.

{{< simulation title="Prefill, then decode: what goes through the model at each step">}}
<div id="pd-widget" style="background:transparent;color:#1e293b;border-radius:10px;padding:16px;font-family:ui-sans-serif,system-ui,sans-serif;">
<style>
#pd-widget .pd-seq{display:flex;flex-wrap:wrap;gap:6px;margin-bottom:12px}
#pd-widget .pd-col{display:flex;flex-direction:column;align-items:center;gap:4px;width:64px}
#pd-widget .pd-pill{width:64px;height:40px;border-radius:8px;display:flex;align-items:center;justify-content:center;font-size:13px;font-weight:700;box-sizing:border-box}
#pd-widget .pd-cap{font-size:11px;color:#64748b;height:14px}
#pd-widget .k-p{border:2px solid #4f46e5}
#pd-widget .k-g{border:2px solid #059669}
#pd-widget .s-proc.k-p{background:#a5b4fc;color:#1e1b4b}
#pd-widget .s-proc.k-g{background:#6ee7b7;color:#064e3b}
#pd-widget .s-cache{background:#f1f5f9;color:#475569}
#pd-widget .s-out{background:#fef3c7;border-color:#d97706;color:#78350f}
#pd-widget .s-future{background:transparent;border:2px dashed #cbd5e1;color:#94a3b8}
#pd-widget .pd-legend{display:flex;flex-wrap:wrap;gap:14px;font-size:12px;color:#475569;margin-bottom:12px}
#pd-widget .pd-legend i{display:inline-block;width:12px;height:12px;border-radius:3px;vertical-align:-1px;margin-right:5px;box-sizing:border-box}
#pd-widget .pd-l1{font-size:15px;font-weight:700;margin-bottom:4px}
#pd-widget .pd-l2{font-size:14px;color:#334155;margin-bottom:12px}
#pd-widget .pd-bar{height:6px;background:#e2e8f0;border-radius:3px;overflow:hidden;margin-bottom:12px}
#pd-widget .pd-bar span{display:block;height:100%;background:#059669;transition:width .2s}
#pd-widget button{padding:8px 14px;font-size:14px;font-weight:600;border-radius:8px;border:1px solid #cbd5e1;background:#fff;color:#334155;cursor:pointer;margin-right:8px}
#pd-widget button.pd-primary{background:#059669;border-color:#059669;color:#fff}
#pd-widget button:disabled{opacity:.45;cursor:not-allowed}
</style>
<div class="pd-seq" id="pd-seq"></div>
<div class="pd-legend">
<span><i style="background:#a5b4fc;border:2px solid #4f46e5"></i>prompt token being processed</span>
<span><i style="background:#6ee7b7;border:2px solid #059669"></i>new input token</span>
<span><i style="background:#f1f5f9;border:2px solid #94a3b8"></i>already in the KV cache</span>
<span><i style="background:#fef3c7;border:2px solid #d97706"></i>predicted this step</span>
</div>
<div class="pd-l1" id="pd-l1"></div>
<div class="pd-l2" id="pd-l2"></div>
<div class="pd-bar"><span id="pd-bar" style="width:0%"></span></div>
<div>
<button type="button" id="pd-prev">← Back</button>
<button type="button" class="pd-primary" id="pd-next">Next step →</button>
<button type="button" id="pd-reset">Reset</button>
</div>
<script>
(function () {
  var T = ['The','quick','brown','fox','jumps','over','the','lazy','dog'];
  var P = 4, MAX = 4, s = 0;
  var seq = document.getElementById('pd-seq');
  var l1 = document.getElementById('pd-l1'), l2 = document.getElementById('pd-l2');
  var bar = document.getElementById('pd-bar');
  var prev = document.getElementById('pd-prev'), next = document.getElementById('pd-next');
  function state(i) {
    if (i === P + s) return 'out';
    if (i > P + s) return 'future';
    if (s === 0) return 'proc';
    if (i === P + s - 1) return 'proc';
    return 'cache';
  }
  var caps = { proc: 'input', cache: 'cached', out: 'predicted', future: '' };
  function render() {
    seq.innerHTML = '';
    for (var i = 0; i < T.length; i++) {
      var st = state(i);
      var col = document.createElement('div'); col.className = 'pd-col';
      var pill = document.createElement('div');
      pill.className = 'pd-pill ' + (i < P ? 'k-p' : 'k-g') + ' s-' + st;
      pill.textContent = T[i];
      var cap = document.createElement('div'); cap.className = 'pd-cap'; cap.textContent = caps[st];
      col.appendChild(pill); col.appendChild(cap); seq.appendChild(col);
    }
    if (s === 0) {
      l1.textContent = 'Prefill: all 4 prompt tokens go through the model together.';
      l2.textContent = 'Cache: 0 → 4 entries. Predicted: “' + T[P] + '”.';
    } else {
      l1.textContent = 'Decode step ' + s + ': only “' + T[P + s - 1] + '” goes through the model.';
      l2.textContent = 'It attends to the ' + (P + s - 1) + ' cached entries plus itself. Cache: ' + (P + s - 1) + ' → ' + (P + s) + ' entries. Predicted: “' + T[P + s] + '”.';
    }
    bar.style.width = (s / MAX * 100) + '%';
    prev.disabled = s === 0;
    next.disabled = s === MAX;
    next.textContent = s === MAX ? 'Done' : 'Next step →';
  }
  next.addEventListener('click', function () { if (s < MAX) { s++; render(); } });
  prev.addEventListener('click', function () { if (s > 0) { s--; render(); } });
  document.getElementById('pd-reset').addEventListener('click', function () { s = 0; render(); });
  render();
})();
</script>
</div>

{{< /simulation >}}

After prefill the cache holds 4 entries, one per prompt token. Each decode step adds one more, so after 4 decode steps it holds 8. The token written last is never in the cache yet, because it only goes through the model, and gets its entry, on the following step.

The work per step is roughly constant: one token through the model, plus attention over a cache that grows by one entry. That is why decode stays cheap per step, and why memory is its main cost. Every step reads the cache, and the cache gets bigger all the time.

## Timeline of one request

Here is a whole request on a time axis. It shows one long prefill step followed by many short decode steps.

{{< simulation title="Wall-clock view of one request: one long prefill step, then many short decode steps">}}
<svg viewBox="0 0 760 240" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="m3" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="240" rx="10" fill="none"/>
<line x1="40" y1="170" x2="740" y2="170" stroke="#cbd5e1" stroke-width="1.5"/>
<rect x="40" y="70" width="200" height="100" rx="6" fill="#4f46e5" stroke="#4338ca" stroke-width="1"/>
<text x="140" y="112" font-size="17" font-weight="700" fill="#ffffff" text-anchor="middle">PREFILL</text>
<text x="140" y="134" font-size="12" font-weight="400" fill="#e0e7ff" text-anchor="middle">2,000 prompt tokens</text>
<text x="140" y="150" font-size="12" font-weight="400" fill="#e0e7ff" text-anchor="middle">in one pass</text>
<rect x="250" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="270" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="290" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="310" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="330" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="350" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="370" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="390" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="410" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="430" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<text x="470" y="160" font-size="18" font-weight="700" fill="#475569" text-anchor="middle">…</text>
<rect x="500" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<rect x="520" y="130" width="14" height="40" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.2"/>
<line x1="250" y1="118" x2="534" y2="118" stroke="#059669" stroke-width="1.5"/>
<text x="392" y="108" font-size="13" font-weight="700" fill="#047857" text-anchor="middle">DECODE: 199 steps × ≈ 25 ms ≈ 5 s</text>
<line x1="240" y1="40" x2="240" y2="170" stroke="#475569" stroke-width="1.2" stroke-dasharray="3 3"/>
<text x="246" y="52" font-size="12" font-weight="700" fill="#1f2937" text-anchor="start">TTFT ≈ 0.5 s: the first token appears</text>
<line x1="560" y1="100" x2="560" y2="170" stroke="#475569" stroke-width="1.2" stroke-dasharray="3 3"/>
<text x="40" y="192" font-size="12" font-weight="400" fill="#475569" text-anchor="middle">t = 0</text>
<text x="240" y="192" font-size="12" font-weight="400" fill="#475569" text-anchor="middle">0.5 s</text>
<text x="560" y="192" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">≈ 5.5 s: reply complete</text>
<text x="578" y="96" font-size="12" font-weight="600" fill="#4338ca" text-anchor="start">Prefill: one long step.</text>
<text x="578" y="114" font-size="12" font-weight="600" fill="#047857" text-anchor="start">Decode: many short steps.</text>
<text x="380" y="224" font-size="12" font-weight="400" fill="#64748b" text-anchor="middle" font-style="italic">Illustrative numbers for a 200-token reply. Not to scale.</text>
</svg>
{{< /simulation >}}

In this diagram:

- **TTFT lives inside the prefill bar.** The model cannot emit a first token until it has read the whole prompt. A 1-token prompt feels instant, and a 10,000-token prompt does not.
- **Decode steps are all about the same length.** Step time creeps up as the cache grows. It climbs faster once the cache gets large enough to compete with the model weights for GPU memory.
- **The phases add up.** Total time is the prefill time plus the decode steps. A faster prefill lowers TTFT and leaves the decode speed alone. A faster decode step speeds up streaming and leaves TTFT alone.

## Side by side

{{< simulation title="Prefill vs decode: same model, different shape of work">}}
<svg viewBox="0 0 760 336" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block; font-family: ui-sans-serif, system-ui, sans-serif;">
<defs><marker id="m4" viewBox="0 0 10 10" refX="9" refY="5" markerUnits="userSpaceOnUse" markerWidth="9" markerHeight="9" orient="auto"><path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/></marker></defs>
<rect width="760" height="336" rx="10" fill="none"/>
<rect x="20" y="16" width="350" height="36" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/>
<text x="195" y="40" font-size="14" font-weight="700" fill="#4338ca" text-anchor="middle">PREFILL: whole prompt, in parallel</text>
<rect x="390" y="16" width="350" height="36" rx="6" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/>
<text x="565" y="40" font-size="14" font-weight="700" fill="#047857" text-anchor="middle">DECODE: one token per step</text>
<rect x="40" y="84" width="70" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="75.0" y="105.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">The</text>
<line x1="75" y1="120" x2="195" y2="158" stroke="#4f46e5" stroke-width="2" marker-end="url(#m4)"/>
<rect x="116" y="84" width="70" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="151.0" y="105.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">quick</text>
<line x1="151" y1="120" x2="195" y2="158" stroke="#4f46e5" stroke-width="2" marker-end="url(#m4)"/>
<rect x="192" y="84" width="70" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="227.0" y="105.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">brown</text>
<line x1="227" y1="120" x2="195" y2="158" stroke="#4f46e5" stroke-width="2" marker-end="url(#m4)"/>
<rect x="268" y="84" width="70" height="34" rx="6" fill="#e0e7ff" stroke="#4f46e5" stroke-width="1.5"/><text x="303.0" y="105.5" font-size="13" font-weight="600" fill="#1f2937" text-anchor="middle">fox</text>
<line x1="303" y1="120" x2="195" y2="158" stroke="#4f46e5" stroke-width="2" marker-end="url(#m4)"/>
<text x="40" y="76" font-size="11" font-weight="400" fill="#64748b" text-anchor="start">input</text>
<rect x="60" y="160" width="270" height="56" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="2"/><text x="195.0" y="193.2" font-size="15" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<text x="195" y="207" font-size="12" font-weight="400" fill="#475569" text-anchor="middle">4 tokens, one batched pass</text>
<line x1="160" y1="216" x2="120" y2="246" stroke="#475569" stroke-width="1.5" marker-end="url(#m4)"/>
<line x1="230" y1="216" x2="280" y2="246" stroke="#475569" stroke-width="1.5" marker-end="url(#m4)"/>
<rect x="30" y="248" width="166" height="40" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="113.0" y="272.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">writes 4 cache entries</text>
<rect x="210" y="248" width="140" height="40" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="1.5"/><text x="280.0" y="272.2" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">predicts “jumps”</text>
<text x="195" y="316" font-size="13" font-weight="700" fill="#4338ca" text-anchor="middle" font-style="italic">limited by compute</text>
<rect x="410" y="84" width="52" height="34" rx="6" fill="#f1f5f9" stroke="#059669" stroke-width="1.5"/><text x="436.0" y="104.8" font-size="11" font-weight="600" fill="#475569" text-anchor="middle">The</text>
<rect x="466" y="84" width="52" height="34" rx="6" fill="#f1f5f9" stroke="#059669" stroke-width="1.5"/><text x="492.0" y="104.8" font-size="11" font-weight="600" fill="#475569" text-anchor="middle">quick</text>
<rect x="522" y="84" width="52" height="34" rx="6" fill="#f1f5f9" stroke="#059669" stroke-width="1.5"/><text x="548.0" y="104.8" font-size="11" font-weight="600" fill="#475569" text-anchor="middle">brown</text>
<rect x="578" y="84" width="52" height="34" rx="6" fill="#f1f5f9" stroke="#059669" stroke-width="1.5"/><text x="604.0" y="104.8" font-size="11" font-weight="600" fill="#475569" text-anchor="middle">fox</text>
<rect x="644" y="84" width="66" height="34" rx="6" fill="#6ee7b7" stroke="#059669" stroke-width="2.5"/><text x="677.0" y="105.5" font-size="13" font-weight="700" fill="#064e3b" text-anchor="middle">jumps</text>
<text x="410" y="76" font-size="11" font-weight="400" fill="#64748b" text-anchor="start">read from the cache</text>
<text x="710" y="76" font-size="11" font-weight="400" fill="#64748b" text-anchor="end">new input</text>
<line x1="520" y1="120" x2="540" y2="158" stroke="#059669" stroke-width="2" marker-end="url(#m4)"/>
<line x1="677" y1="120" x2="620" y2="158" stroke="#059669" stroke-width="2" marker-end="url(#m4)"/>
<rect x="430" y="160" width="270" height="56" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="2"/><text x="565.0" y="193.2" font-size="15" font-weight="600" fill="#1f2937" text-anchor="middle">Transformer</text>
<text x="565" y="207" font-size="12" font-weight="400" fill="#475569" text-anchor="middle">1 new token, reads the cache</text>
<line x1="530" y1="216" x2="500" y2="246" stroke="#475569" stroke-width="1.5" marker-end="url(#m4)"/>
<line x1="600" y1="216" x2="650" y2="246" stroke="#475569" stroke-width="1.5" marker-end="url(#m4)"/>
<rect x="410" y="248" width="150" height="40" rx="6" fill="#f1f5f9" stroke="#475569" stroke-width="1.5"/><text x="485.0" y="272.2" font-size="12" font-weight="600" fill="#1f2937" text-anchor="middle">adds 1 cache entry</text>
<rect x="580" y="248" width="140" height="40" rx="6" fill="#fef3c7" stroke="#d97706" stroke-width="1.5"/><text x="650.0" y="272.2" font-size="12" font-weight="700" fill="#1f2937" text-anchor="middle">predicts “over”</text>
<text x="565" y="316" font-size="13" font-weight="700" fill="#047857" text-anchor="middle" font-style="italic">limited by memory bandwidth</text>
</svg>
{{< /simulation >}}

| | Prefill | Decode |
| --- | --- | --- |
| Tokens per pass | The whole prompt | One |
| Shape of the work | A few large matrix multiplications | Many small, repeated steps |
| Limited by | Compute | Memory bandwidth |
| KV cache | Writes one entry per prompt token | Appends one entry per step |
| Latency it controls | Time to first token | Time per output token |

## The metrics

Three numbers show up on every LLM serving dashboard.

**TTFT (time to first token)** is the time from sending the request to seeing the first output token. To a first approximation it is the prefill time, plus any time the request spent waiting in a queue. It grows with prompt length. At very long contexts it grows faster than linearly, because attention cost grows with the square of the sequence length.

**Tokens per second (TPS)** is the streaming speed after the first token, about `1 / decode step time`. For a given model and hardware it is mostly independent of prompt length, though it does drift down at very long contexts as the cache grows.

**Total response time** is TTFT plus the remaining decode steps: `TTFT + (output tokens − 1) × step time`. The first output token comes out of prefill, so decode only has to produce the rest. A server can have a fast TTFT and a slow TPS, or the reverse, because the two metrics come from different phases. For a chatbot, TTFT is what the user stares at. For throughput per GPU, decode step time matters more.

## Why the two phases hit different limits

Both phases run on the same GPU, and they stress it in different ways.

Prefill is **compute-bound**. A 1,000-token prompt multiplies each weight matrix by 1,000 token vectors at once, so every weight the GPU loads from memory gets reused 1,000 times. The GPU's arithmetic units are the bottleneck, and batching more requests together does not help once they are saturated.

Decode is **memory-bandwidth-bound**. One step multiplies the same weights by a single token vector, so each weight loaded from memory is used once and then discarded. The step also reads the whole KV cache. Most of the step time goes to moving bytes, with little arithmetic to do on them. The way to get more out of the GPU is to run many requests' decode steps together, so one load of the weights serves all of them. This is what continuous batching does.

The cache is the second memory cost. It lives in GPU memory next to the model weights, and every request in flight has its own. Longer contexts mean bigger caches, fewer concurrent requests, and slower steps. A very long context is a problem mostly because of memory, and arithmetic is secondary. The exact size depends on the model, and I don't cover the formula here

## What this explains in practice

- **Continuous batching**: the server mixes decode steps from many requests into one GPU batch. It works because decode steps from different requests are independent, and batching them amortizes the weight loading.
- **Prompt caching**: if many requests share a long prefix, such as a system prompt, the server can keep that prefix's KV cache and reuse it. This saves prefill work, so it lowers TTFT.
- **Speculative decoding**: a small draft model proposes several tokens, and the large model checks them in one pass, which looks like a short prefill. Accepted tokens skip individual decode steps, so TPS rises, and the output distribution stays the same.
- **Separate prefill and decode machines**: large deployments often give each phase its own pool of GPUs. Prefill machines are tuned for compute, decode machines for memory, and a scheduler hands requests from one to the other.

You can feel both phases from the outside. If a chat takes twenty seconds before the first character appears, prefill is working through a huge prompt. If a long conversation streams more slowly as it grows, the cache is getting large and each decode step is reading more memory.


> [!NOTE] A Note on AI Assistance
> Yes, I used AI to help write this blog. But the research, the experiments, figuring out what to cover, deciding what should come before what, and connecting the ideas were done by me.
>AI helped me put my thoughts into better words. The thinking and learning behind the blog are mine.