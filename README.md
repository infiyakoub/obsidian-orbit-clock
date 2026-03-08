# Obsidian Orbit Clock

An orbit-style clock for Obsidian built using DataviewJS and SVG. Inspired by orbit clock widgets.

## Requirements
- Obsidian
- Dataview plugin

## Usage

Create a note and paste the following:

```dataviewjs
/*******************************************************
Orbit Clock — Obsidian DataviewJS SVG
- Ring 2: 1 rotation / minute
  - Satellite ring on the minute planet: dot rotates once per second
- Ring 3: 1 rotation / hour
- Ring 4 (outer): 1 rotation / day

Top-start guarantee:
- All markers are placed graphically at TOP (12 o’clock) in their base position.
- All rotations use angle = progress * 360, where progress is calculated from
  absolute time between local “startOfX” and “nextX”.

Animation modes:
- cfg.animation = "raf"      (smooth, recommended)
- cfg.animation = "interval" (capped framerate, lower CPU usage)
- cfg.animation = "step"     (quantized tick-look, still accurate)
*******************************************************/

const cfg = {
  size: 340,               // px (SVG viewBox and default render size)
  padding: 18,             // px (space to the edge)
  ringGap: 44,             // px between rings
  stroke: 2,               // px
  color: "currentColor",   // inherits from theme
  background: "transparent",

  // Visual
  opacityRing: 0.95,
  opacityTicks: 0.80,
  opacityText: 0.95,
  nonScalingStroke: false, // true if you want to avoid stroke scaling with CSS scale

  // Markers
  centerDot: 12,
  ring2Dot: 7.5,     // minute planet
  ring3Dot: 12,      // hour
  ring4Dot: 14,      // day (outer)

  // Satellite (attached to the ring 2 dot)
  satRingRadius: 14,
  satDotRadius: 4,

  // Ticks (static clock-face layer)
  ticks: {
    show: true,
    count: 12,          // 12 time indicators
    majorEvery: 3,      // 0, 3, 6, 9 become “major”
    lenMinor: 10,
    lenMajor: 18,
    stroke: 2
  },

  // Time text
  timeText: {
    show: true,
    fontFamily: "system-ui, -apple-system, Segoe UI, Inter, Roboto, Arial",
    fontSize: 16,
    yOffset: -25,        // px (up/down relative to center)
    showSeconds: true,
    hour12: true,
    showAmPm: true,
    tabularNums: true
  },

  // Day cycle:
  // "calendar" = local midnight → next local midnight (DST-correct)
  // "fixed24h" = exact 24h (86400000ms) from local midnight
  //              (may differ from next local midnight on DST days)
  dayMode: "calendar",

  // Animation
  animation: "raf",      // "raf" | "interval" | "step"
  intervalMs: 40,        // used if animation = "interval"

  // Step / “tick” look (used if animation = "step")
  quantize: {
    // number of steps per cycle: higher = smoother, lower = more tick-like
    second: 24,          // satellite dot (1 second cycle)
    minute: 60,          // ring 2 (1 minute cycle)
    hour: 60,            // ring 3 (1 hour cycle)
    day: 1440            // ring 4 (1 day cycle) -> 1440 = 1/minute
  }
};

// ---------- Safe cleanup between re-renders ----------
const host = dv.container;
const GLOBAL_KEY = "__orbitClockInstances";
window[GLOBAL_KEY] ??= new WeakMap();
const instances = window[GLOBAL_KEY];

const prev = instances.get(host);
if (prev?.stop) prev.stop();

// ---------- Helpers ----------
const svgNS = "http://www.w3.org/2000/svg";

function el(name, attrs = {}) {
  const n = document.createElementNS(svgNS, name);
  for (const [k, v] of Object.entries(attrs)) n.setAttribute(k, String(v));
  return n;
}

function setAttrs(node, attrs = {}) {
  for (const [k, v] of Object.entries(attrs)) node.setAttribute(k, String(v));
}

function pad2(n) { return String(n).padStart(2, "0"); }

function formatTime(d) {
  const h24 = d.getHours();
  const m = d.getMinutes();
  const s = d.getSeconds();

  if (!cfg.timeText.hour12) {
    return cfg.timeText.showSeconds
      ? `${pad2(h24)}:${pad2(m)}:${pad2(s)}`
      : `${pad2(h24)}:${pad2(m)}`;
  }

  const am = h24 < 12;
  const h12 = ((h24 + 11) % 12) + 1;
  const core = cfg.timeText.showSeconds
    ? `${h12}:${pad2(m)}:${pad2(s)}`
    : `${h12}:${pad2(m)}`;

  if (!cfg.timeText.showAmPm) return core;
  return `${core} ${am ? "AM" : "PM"}`;
}

function quantize01(p, steps) {
  if (!steps || steps <= 0) return p;
  return Math.round(p * steps) / steps;
}

// degFromTop: 0=top, 90=right, 180=bottom, 270=left
function polarFromTop(cx, cy, r, degFromTop) {
  const a = (degFromTop * Math.PI / 180) - (Math.PI / 2);
  return { x: cx + Math.cos(a) * r, y: cy + Math.sin(a) * r };
}

function progressBetween(now, startMs, endMs) {
  const dur = endMs - startMs;
  if (dur <= 0) return 0;
  let p = (now - startMs) / dur;
  // wrap: if p=1 due to an edge frame, bring it back to 0
  p = p - Math.floor(p);
  return p;
}

// Bounds helpers (local time)
function boundsSecond(d) {
  const a = new Date(d);
  a.setMilliseconds(0);
  const b = new Date(a);
  b.setSeconds(a.getSeconds() + 1);
  return [a.getTime(), b.getTime()];
}

function boundsMinute(d) {
  const a = new Date(d);
  a.setSeconds(0, 0);
  const b = new Date(a);
  b.setMinutes(a.getMinutes() + 1);
  return [a.getTime(), b.getTime()];
}

function boundsHour(d) {
  const a = new Date(d);
  a.setMinutes(0, 0, 0);
  const b = new Date(a);
  b.setHours(a.getHours() + 1);
  return [a.getTime(), b.getTime()];
}

function boundsDay(d) {
  const a = new Date(d);
  a.setHours(0, 0, 0, 0);

  if (cfg.dayMode === "fixed24h") {
    const start = a.getTime();
    return [start, start + 86400000];
  }

  const b = new Date(a);
  b.setDate(a.getDate() + 1);
  return [a.getTime(), b.getTime()];
}

// ---------- Build DOM ----------
host.innerHTML = "";
host.classList.add("orbit-clock");
host.style.color = cfg.color;

const svg = el("svg", {
  viewBox: `0 0 ${cfg.size} ${cfg.size}`,
  width: cfg.size,
  height: cfg.size,
  role: "img"
});
svg.style.background = cfg.background;
svg.style.display = "block";

const root = el("g");
svg.appendChild(root);

const cx = cfg.size / 2;
const cy = cfg.size / 2;
const outerR = (cfg.size / 2) - cfg.padding;

const r4 = outerR;                 // day
const r3 = outerR - cfg.ringGap;   // hour
const r2 = outerR - cfg.ringGap * 2; // minute
const r1 = outerR - cfg.ringGap * 3; // previously inner fixed, now not rendered

if (r1 <= 0) {
  const warn = document.createElement("div");
  warn.textContent = "Orbit Clock: size/ringGap/padding results in a negative inner radius. Increase size or reduce ringGap.";
  host.appendChild(warn);
  throw new Error("Orbit Clock config invalid: inner radius <= 0");
}

function ringCircle(r) {
  const c = el("circle", {
    cx, cy, r,
    fill: "none",
    stroke: cfg.color,
    "stroke-width": cfg.stroke,
    "stroke-linecap": "round",
    opacity: cfg.opacityRing
  });
  if (cfg.nonScalingStroke) c.setAttribute("vector-effect", "non-scaling-stroke");
  return c;
}

function markerDotAtTop(r, dotR) {
  const m = el("circle", {
    cx: cx,
    cy: cy - r,        // TOP at base position
    r: dotR,
    fill: cfg.color,
    opacity: 1
  });
  return m;
}

// Static layer: ticks (must NOT rotate)
if (cfg.ticks.show) {
  const gTicks = el("g");
  // place ticks visually between outer ring and ring 3
  const tickOuter = r3 + 10;
  for (let i = 0; i < cfg.ticks.count; i++) {
    const isMajor = (i % cfg.ticks.majorEvery) === 0;
    const len = isMajor ? cfg.ticks.lenMajor : cfg.ticks.lenMinor;
    const deg = (i * 360) / cfg.ticks.count; // 0 = top

    const p1 = polarFromTop(cx, cy, tickOuter, deg);
    const p2 = polarFromTop(cx, cy, tickOuter - len, deg);

    const line = el("line", {
      x1: p1.x, y1: p1.y, x2: p2.x, y2: p2.y,
      stroke: cfg.color,
      "stroke-width": cfg.ticks.stroke,
      "stroke-linecap": "round",
      opacity: isMajor ? cfg.opacityTicks : (cfg.opacityTicks * 0.9)
    });
    if (cfg.nonScalingStroke) line.setAttribute("vector-effect", "non-scaling-stroke");
    gTicks.appendChild(line);
  }
  root.appendChild(gTicks);
}

// Ring 4 (day) — rotating group
const g4 = el("g");
g4.appendChild(ringCircle(r4));
g4.appendChild(markerDotAtTop(r4, cfg.ring4Dot));
root.appendChild(g4);

// Ring 3 (hour) — rotating group
const g3 = el("g");
g3.appendChild(ringCircle(r3));
g3.appendChild(markerDotAtTop(r3, cfg.ring3Dot));
root.appendChild(g3);

// Ring 2 (minute) — rotating group + satellite on dot
const g2 = el("g");
g2.appendChild(ringCircle(r2));

// minute planet (dot) at top
const minuteDot = markerDotAtTop(r2, cfg.ring2Dot);
g2.appendChild(minuteDot);

// satellite: centered on the minute dot (same top position in base state)
const satAnchorX = cx;
const satAnchorY = cy - r2;

const satGroup = el("g", { transform: `translate(${satAnchorX} ${satAnchorY})` });

// satellite ring (outline)
const satRing = el("circle", {
  cx: 0, cy: 0,
  r: cfg.satRingRadius,
  fill: "none",
  stroke: cfg.color,
  "stroke-width": cfg.stroke,
  opacity: cfg.opacityRing
});
if (cfg.nonScalingStroke) satRing.setAttribute("vector-effect", "non-scaling-stroke");
satGroup.appendChild(satRing);

// satOrbit wrapper: dot is placed at top and rotates around (0,0)
const satOrbit = el("g");
const satDot = el("circle", {
  cx: 0,
  cy: -cfg.satRingRadius, // TOP at base position
  r: cfg.satDotRadius,
  fill: cfg.color,
  opacity: 1
});
satOrbit.appendChild(satDot);
satGroup.appendChild(satOrbit);

g2.appendChild(satGroup);
root.appendChild(g2);

// Center dot
const center = el("circle", {
  cx, cy,
  r: cfg.centerDot,
  fill: cfg.color,
  opacity: 1
});
root.appendChild(center);

// Time text
let timeTextNode = null;
if (cfg.timeText.show) {
  timeTextNode = el("text", {
    x: cx,
    y: cy + cfg.timeText.yOffset,
    "text-anchor": "middle",
    "dominant-baseline": "middle",
    fill: cfg.color,
    "font-size": cfg.timeText.fontSize,
    "font-family": cfg.timeText.fontFamily,
    opacity: cfg.opacityText
  });
  if (cfg.timeText.tabularNums) {
    timeTextNode.setAttribute("style", "font-variant-numeric: tabular-nums;");
  }
  root.appendChild(timeTextNode);
}

host.appendChild(svg);

// ---------- Update ----------
function update(now = Date.now()) {
  const d = new Date(now);

  const [s0, s1] = boundsSecond(d);
  const [m0, m1] = boundsMinute(d);
  const [h0, h1] = boundsHour(d);
  const [d0, d1] = boundsDay(d);

  let pSec = progressBetween(now, s0, s1); // satellite
  let pMin = progressBetween(now, m0, m1); // ring 2
  let pHr  = progressBetween(now, h0, h1); // ring 3
  let pDay = progressBetween(now, d0, d1); // ring 4

  // Step / “tick” look
  if (cfg.animation === "step") {
    pSec = quantize01(pSec, cfg.quantize.second);
    pMin = quantize01(pMin, cfg.quantize.minute);
    pHr  = quantize01(pHr, cfg.quantize.hour);
    pDay = quantize01(pDay, cfg.quantize.day);
  }

  // 0 = top, clockwise
  const aSec = pSec * 360;
  const aMin = pMin * 360;
  const aHr  = pHr * 360;
  const aDay = pDay * 360;

  // Rings 2/3/4 rotate as groups around center
  g2.setAttribute("transform", `rotate(${aMin} ${cx} ${cy})`);
  g3.setAttribute("transform", `rotate(${aHr} ${cx} ${cy})`);
  g4.setAttribute("transform", `rotate(${aDay} ${cx} ${cy})`);

  // Satellite dot rotates around the minute dot on its own axis
  satOrbit.setAttribute("transform", `rotate(${aSec})`);

  if (timeTextNode) timeTextNode.textContent = formatTime(d);
}

// ---------- Animation driver ----------
let rafId = null;
let intervalId = null;
let stopped = false;

function stop() {
  stopped = true;
  if (rafId !== null) cancelAnimationFrame(rafId);
  if (intervalId !== null) clearInterval(intervalId);
  rafId = null;
  intervalId = null;
}

function loopRAF() {
  if (stopped || !host.isConnected) { stop(); return; }
  update();
  rafId = requestAnimationFrame(loopRAF);
}

update(); // initial paint

if (cfg.animation === "raf" || cfg.animation === "step") {
  rafId = requestAnimationFrame(loopRAF);
} else if (cfg.animation === "interval") {
  intervalId = setInterval(() => {
    if (stopped || !host.isConnected) { stop(); return; }
    update();
  }, cfg.intervalMs);
}

instances.set(host, { stop });
```
