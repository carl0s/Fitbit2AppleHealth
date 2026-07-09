<template>
  <div class="hrc">
    <div class="hrc__stats">
      <div class="hrc__stat">
        <span class="hrc__stat-value">{{ minValue }}</span>
        <span class="hrc__stat-label">min</span>
      </div>
      <div v-if="restingHr" class="hrc__stat">
        <span class="hrc__stat-value">{{ restingHr }}</span>
        <span class="hrc__stat-label">resting</span>
      </div>
      <div class="hrc__stat">
        <span class="hrc__stat-value">{{ maxValue }}</span>
        <span class="hrc__stat-label">max</span>
      </div>
      <span class="hrc__unit">bpm</span>
    </div>

    <div class="hrc__plot">
      <svg
        ref="svgEl"
        class="hrc__svg"
        :viewBox="`0 0 ${W} ${H}`"
        @pointermove="onPointer"
        @pointerdown="onPointer"
        @pointerleave="hover = null"
        @pointercancel="hover = null"
      >
        <!-- gridlines + y ticks -->
        <g v-for="t in yTicks" :key="'y' + t">
          <line :x1="PAD.l" :x2="W - PAD.r" :y1="yFor(t)" :y2="yFor(t)" class="hrc__grid" />
          <text :x="PAD.l - 5" :y="yFor(t) + 3" class="hrc__tick" text-anchor="end">{{ t }}</text>
        </g>
        <!-- x ticks (hours of day) -->
        <text
          v-for="t in X_TICKS"
          :key="'x' + t.m"
          :x="xFor(t.m)"
          :y="H - 3"
          class="hrc__tick"
          text-anchor="middle"
        >{{ t.label }}</text>
        <!-- resting HR reference -->
        <template v-if="restingY !== null">
          <line :x1="PAD.l" :x2="W - PAD.r" :y1="restingY" :y2="restingY" class="hrc__resting" />
          <text :x="W - PAD.r" :y="restingY - 3" class="hrc__resting-label" text-anchor="end">resting</text>
        </template>
        <!-- area wash + line -->
        <path v-if="areaPath" :d="areaPath" class="hrc__area" />
        <path v-if="linePath" :d="linePath" class="hrc__line" />
        <!-- peak marker (direct label on the extreme only) -->
        <template v-if="maxPoint && !hover">
          <circle :cx="xFor(maxPoint.minute)" :cy="yFor(maxPoint.value)" r="4" class="hrc__dot" />
          <text :x="maxLabelX" :y="yFor(maxPoint.value) - 8" class="hrc__peak" text-anchor="middle">{{ maxPoint.value }}</text>
        </template>
        <!-- crosshair -->
        <template v-if="hover">
          <line :x1="hover.x" :x2="hover.x" :y1="PAD.t" :y2="H - PAD.b" class="hrc__cross" />
          <circle :cx="hover.x" :cy="yFor(hover.value)" r="4" class="hrc__dot" />
        </template>
      </svg>
      <div v-if="hover" class="hrc__tip" :style="{ left: tipLeft }">
        {{ hover.label }} · {{ hover.value }} bpm
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';

interface Sample { time: string; value: number }

const props = defineProps<{
  samples: Sample[];
  restingHr?: number | null;
}>();

const W = 360;
const H = 150;
const PAD = { l: 30, r: 8, t: 14, b: 16 };

const X_TICKS = [
  { m: 0, label: '00' },
  { m: 360, label: '06' },
  { m: 720, label: '12' },
  { m: 1080, label: '18' },
  { m: 1440, label: '24' },
];

const svgEl = ref<SVGSVGElement | null>(null);
const hover = ref<{ x: number; value: number; label: string } | null>(null);

const raw = computed(() =>
  props.samples
    .map((s) => {
      const [h, m] = s.time.split(':').map(Number);
      return { minute: h * 60 + m, value: s.value };
    })
    .filter((p) => !isNaN(p.minute) && p.value > 0)
    .sort((a, b) => a.minute - b.minute)
);

// 5-minute averaged buckets keep the path light (max ~290 points)
const points = computed(() => {
  const buckets = new Map<number, { sum: number; n: number }>();
  for (const p of raw.value) {
    const b = Math.floor(p.minute / 5) * 5;
    const cur = buckets.get(b) || { sum: 0, n: 0 };
    cur.sum += p.value;
    cur.n++;
    buckets.set(b, cur);
  }
  return [...buckets.entries()]
    .map(([minute, { sum, n }]) => ({ minute, value: sum / n }))
    .sort((a, b) => a.minute - b.minute);
});

const minValue = computed(() => (raw.value.length ? Math.min(...raw.value.map((p) => p.value)) : 0));
const maxValue = computed(() => (raw.value.length ? Math.max(...raw.value.map((p) => p.value)) : 0));
const maxPoint = computed(() => {
  if (!raw.value.length) return null;
  return raw.value.find((p) => p.value === maxValue.value) || null;
});

const yLo = computed(() => {
  const base = Math.min(minValue.value, props.restingHr || minValue.value);
  return Math.floor((base - 5) / 10) * 10;
});
const yHi = computed(() => Math.max(yLo.value + 10, Math.ceil((maxValue.value + 5) / 10) * 10));
const yTicks = computed(() => {
  const mid = Math.round((yLo.value + yHi.value) / 2 / 5) * 5;
  return mid > yLo.value && mid < yHi.value ? [yLo.value, mid, yHi.value] : [yLo.value, yHi.value];
});

const plotW = W - PAD.l - PAD.r;
const xFor = (minute: number) => PAD.l + (minute / 1440) * plotW;
const yFor = (v: number) =>
  PAD.t + (1 - (v - yLo.value) / (yHi.value - yLo.value)) * (H - PAD.t - PAD.b);

const linePath = computed(() =>
  points.value
    .map((p, i) => `${i ? 'L' : 'M'}${xFor(p.minute).toFixed(1)},${yFor(p.value).toFixed(1)}`)
    .join(' ')
);
const areaPath = computed(() => {
  if (!points.value.length) return '';
  const first = points.value[0];
  const last = points.value[points.value.length - 1];
  const base = H - PAD.b;
  return `${linePath.value} L${xFor(last.minute).toFixed(1)},${base} L${xFor(first.minute).toFixed(1)},${base} Z`;
});

const restingY = computed(() => {
  const r = props.restingHr;
  if (!r || r < yLo.value || r > yHi.value) return null;
  return yFor(r);
});

const maxLabelX = computed(() => {
  if (!maxPoint.value) return 0;
  return Math.min(Math.max(xFor(maxPoint.value.minute), PAD.l + 12), W - PAD.r - 12);
});

const tipLeft = computed(() => {
  if (!hover.value) return '50%';
  const pct = (hover.value.x / W) * 100;
  return `${Math.min(Math.max(pct, 16), 84)}%`;
});

function onPointer(e: PointerEvent) {
  if (!svgEl.value || !points.value.length) return;
  const rect = svgEl.value.getBoundingClientRect();
  const xSvg = ((e.clientX - rect.left) / rect.width) * W;
  const minute = ((xSvg - PAD.l) / plotW) * 1440;
  let best = points.value[0];
  for (const p of points.value) {
    if (Math.abs(p.minute - minute) < Math.abs(best.minute - minute)) best = p;
  }
  const hh = String(Math.floor(best.minute / 60)).padStart(2, '0');
  const mm = String(best.minute % 60).padStart(2, '0');
  hover.value = { x: xFor(best.minute), value: Math.round(best.value), label: `${hh}:${mm}` };
}
</script>

<style scoped>
.hrc__stats {
  display: flex;
  align-items: baseline;
  gap: 16px;
  padding: 2px 4px 10px;
}
.hrc__stat {
  display: flex;
  align-items: baseline;
  gap: 5px;
}
.hrc__stat-value {
  font-family: var(--ion-font-family);
  font-size: 16px;
  font-weight: 700;
  color: var(--ion-text-color);
  letter-spacing: -0.02em;
}
.hrc__stat-label {
  font-family: var(--font-secondary);
  font-size: 9px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--ion-color-medium);
}
.hrc__unit {
  margin-left: auto;
  font-family: var(--font-secondary);
  font-size: 9px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--ion-color-medium);
}

.hrc__plot {
  position: relative;
}
.hrc__svg {
  display: block;
  width: 100%;
  touch-action: pan-y;
}
.hrc__grid {
  stroke: var(--pulso-viz-grid);
  stroke-width: 1;
}
.hrc__tick {
  font-family: var(--ion-font-family);
  font-size: 8.5px;
  fill: var(--ion-color-medium);
}
.hrc__line {
  fill: none;
  stroke: var(--pulso-accent);
  stroke-width: 2;
  stroke-linejoin: round;
  stroke-linecap: round;
}
.hrc__area {
  fill: var(--pulso-accent);
  opacity: 0.08;
  stroke: none;
}
.hrc__resting {
  stroke: var(--ion-color-medium);
  stroke-width: 1;
  opacity: 0.55;
}
.hrc__resting-label {
  font-family: var(--font-secondary);
  font-size: 8px;
  fill: var(--ion-color-medium);
  text-transform: uppercase;
  letter-spacing: 0.05em;
}
.hrc__dot {
  fill: var(--pulso-accent);
  stroke: var(--ion-card-background);
  stroke-width: 2;
}
.hrc__peak {
  font-family: var(--ion-font-family);
  font-size: 9px;
  font-weight: 700;
  fill: var(--ion-text-color);
}
.hrc__cross {
  stroke: var(--ion-color-medium);
  stroke-width: 1;
  opacity: 0.5;
}
.hrc__tip {
  position: absolute;
  top: 0;
  transform: translateX(-50%);
  padding: 3px 8px;
  border-radius: 6px;
  background: var(--ion-text-color);
  color: var(--ion-background-color);
  font-family: var(--ion-font-family);
  font-size: 10px;
  font-weight: 700;
  white-space: nowrap;
  pointer-events: none;
}
</style>
