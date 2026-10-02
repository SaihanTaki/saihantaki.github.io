---
title: "Simulation Shortcode Examples"
summary: "Demonstrates the simulation shortcode with static, interactive, and SVG/canvas content."
categories: ["Post", "Blog"]
tags: ["shortcode", "simulation", "embed"]
date: 2026-09-20
draft: true
---

The `simulation` shortcode renders inline content inside a consistent themed canvas with a subtle graph-paper grid. It does not use an iframe — the inner HTML is part of the page.

## Static SVG diagram

{{< simulation title="Static diagram: simple graph" height="320px" >}}
<svg viewBox="0 0 600 240" width="100%" xmlns="http://www.w3.org/2000/svg" style="display:block;">
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5"
            markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#475569"/>
    </marker>
  </defs>
  <g font-family="ui-sans-serif, system-ui, sans-serif" font-size="14" fill="#1f2937">
    <circle cx="80"  cy="180" r="28" fill="#dbeafe" stroke="#1d4ed8" stroke-width="2"/>
    <circle cx="300" cy="60"  r="28" fill="#dcfce7" stroke="#15803d" stroke-width="2"/>
    <circle cx="520" cy="180" r="28" fill="#fee2e2" stroke="#b91c1c" stroke-width="2"/>
    <text x="80"  y="184" text-anchor="middle">A</text>
    <text x="300" y="64"  text-anchor="middle">B</text>
    <text x="520" y="184" text-anchor="middle">C</text>
    <line x1="108" y1="172" x2="272" y2="68"  stroke="#475569" stroke-width="2" marker-end="url(#arrow)"/>
    <line x1="328" y1="68"  x2="492" y2="172" stroke="#475569" stroke-width="2" marker-end="url(#arrow)"/>
    <line x1="108" y1="180" x2="492" y2="180" stroke="#475569" stroke-width="2" stroke-dasharray="6 4" marker-end="url(#arrow)"/>
  </g>
</svg>
{{< /simulation >}}

## Interactive: slider-driven numeric readout

{{< simulation title="Interactive: a slider" >}}
<div style="display:flex; flex-direction:column; gap:0.75rem; align-items:flex-start;">
  <label for="sim-slider" style="font-weight:600; color:#1f2937;">Value</label>
  <input id="sim-slider" type="range" min="0" max="100" value="50"
         style="width:100%; max-width:420px;">
  <output for="sim-slider" id="sim-slider-out"
          style="font-family:ui-monospace, SFMono-Regular, Menlo, monospace;
                 background:#fff; padding:4px 8px; border-radius:6px;
                 border:1px solid #cbd5e1; color:#0f172a;">50</output>
  <script>
    (function () {
      var slider = document.getElementById('sim-slider');
      var out    = document.getElementById('sim-slider-out');
      if (slider && out) {
        slider.addEventListener('input', function () {
          out.textContent = slider.value;
        });
      }
    })();
  </script>
</div>
{{< /simulation >}}

## Canvas-based animation

{{< simulation title="Canvas: bouncing dot" height="260px" >}}
<canvas id="sim-canvas" width="600" height="200"
        style="display:block; width:100%; max-width:600px;"></canvas>
<script>
  (function () {
    var c   = document.getElementById('sim-canvas');
    if (!c) return;
    var ctx = c.getContext('2d');
    var x = 20, y = 100, vx = 2, vy = 1.4;
    function step() {
      x += vx; y += vy;
      if (x < 10 || x > c.width  - 10) vx *= -1;
      if (y < 10 || y > c.height - 10) vy *= -1;
      ctx.clearRect(0, 0, c.width, c.height);
      ctx.fillStyle = '#1d4ed8';
      ctx.beginPath();
      ctx.arc(x, y, 10, 0, Math.PI * 2);
      ctx.fill();
      requestAnimationFrame(step);
    }
    step();
  })();
  </script>
{{< /simulation >}}

## Untitleed (no caption)

{{< simulation >}}
<p style="margin:0; color:#1f2937;">
  A simulation block without a title. Content remains author-controlled; the
  container just provides the canvas.
</p>
{{< /simulation >}}

## Interactive controls (range, button, script)

{{< simulation title="Gradient Descent" >}}

<div class="controls">
    <label>
        Learning rate
        <input type="range" id="learning-rate"
               min="0.01" max="1" step="0.01" value="0.1">
    </label>

    <button type="button" id="start">Start</button>
    <button type="button" id="reset">Reset</button>
</div>

<canvas id="gradient-descent"></canvas>

<script>
    // Simulation-specific logic.
    (function () {
      var lr = document.getElementById('learning-rate');
      var start = document.getElementById('start');
      var reset = document.getElementById('reset');
      var canvas = document.getElementById('gradient-descent');
      if (!lr || !start || !reset || !canvas) return;
      // Stub: real sim would draw on canvas.
      start.addEventListener('click', function () { canvas.style.border = '2px solid #15803d'; });
      reset.addEventListener('click', function () { canvas.style.border = '1px dashed #94a3b8'; });
    })();
</script>

{{< /simulation >}}
