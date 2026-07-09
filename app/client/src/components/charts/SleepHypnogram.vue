<template>
  <div class="hyp">
    <!-- Stage timeline (hypnogram) -->
    <div v-if="segments.length" class="hyp__plot">
      <svg
        ref="svgEl"
        class="hyp__svg"
        :viewBox="`0 0 ${W} ${H}`"
        @pointermove="onPointer"
        @pointerdown="onPointer"
        @pointerleave="hover = null"
        @pointercancel="hover = null"
      >
        <g v-for="(row, i) in ROWS" :key="row.key">
          <line
            :x1="PAD.l" :x2="W - PAD.r"
            :y1="rowCenter(i)" :y2="rowCenter(i)"
            class="hyp__guide"
          />
          <text :x="PAD.l - 6" :y="rowCenter(i) + 3" class="hyp__row-label" text-anchor="end">{{ row.label }}</text>
        </g>
        <rect
          v-for="(s, i) in segments"
          :key="i"
          :x="s.x" :y="s.y" :width="s.w" :height="BAR"
          rx="3"
          :class="['hyp__seg', 'hyp__fill--' + s.stage]"
        />
        <text
          v-for="(t, i) in xTicks"
          :key="'t' + i"
          :x="t.x" :y="H - 3"
          class="hyp__tick"
          :text-anchor="t.anchor"
        >{{ t.label }}</text>
        <line v-if="hover" :x1="hover.x" :x2="hover.x" :y1="PAD.t" :y2="H - PAD.b" class="hyp__cross" />
      </svg>
      <div v-if="hover" class="hyp__tip" :style="{ left: tipLeft }">
        {{ hover.label }}
      </div>
    </div>

    <!-- Stage composition: stacked bar + legend -->
    <div v-if="totals.length" class="hyp__composition">
      <div class="hyp__bar">
        <div
          v-for="t in totals"
          :key="t.stage"
          :class="'hyp__bar-seg hyp__fill--' + t.stage"
          :style="{ width: t.pct + '%' }"
        />
      </div>
      <div class="hyp__legend">
        <div v-for="t in totals" :key="t.stage" class="hyp__legend-item">
          <span :class="'hyp__swatch hyp__fill--' + t.stage" />
          <span class="hyp__legend-label">{{ t.label }}</span>
          <span class="hyp__legend-value">{{ formatMin(t.minutes) }}</span>
          <span class="hyp__legend-pct">{{ Math.round(t.pct) }}%</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';

interface SleepStage {
  stage: string;
  startTime: string;
  endTime: string;
  seconds: number;
}

const props = defineProps<{
  stages: SleepStage[];
  sleepStart?: string | null;
  sleepEnd?: string | null;
}>();

const ROWS = [
  { key: 'wake', label: 'Awake' },
  { key: 'rem', label: 'REM' },
  { key: 'light', label: 'Light' },
  { key: 'deep', label: 'Deep' },
];

const W = 360;
const PAD = { l: 44, r: 8, t: 6, b: 16 };
const ROW_H = 24;
const BAR = 14;
const H = PAD.t + ROWS.length * ROW_H + PAD.b;

const svgEl = ref<SVGSVGElement | null>(null);
const hover = ref<{ x: number; label: string } | null>(null);

// Fitbit "classic" logs use asleep/restless/awake instead of stage names
function norm(stage: string): string | null {
  const s = (stage || '').toLowerCase();
  if (s === 'deep' || s === 'light' || s === 'rem' || s === 'wake') return s;
  if (s === 'asleep') return 'light';
  if (s === 'restless' || s === 'awake') return 'wake';
  return null;
}

const timed = computed(() => {
  return props.stages
    .map((s) => ({
      stage: norm(s.stage) as string,
      start: new Date(s.startTime).getTime(),
      end: new Date(s.endTime).getTime(),
    }))
    .filter((s) => s.stage && !isNaN(s.start) && !isNaN(s.end) && s.end > s.start);
});

const win = computed(() => {
  let start = props.sleepStart ? new Date(props.sleepStart).getTime() : NaN;
  let end = props.sleepEnd ? new Date(props.sleepEnd).getTime() : NaN;
  if ((isNaN(start) || isNaN(end)) && timed.value.length) {
    start = Math.min(...timed.value.map((s) => s.start));
    end = Math.max(...timed.value.map((s) => s.end));
  }
  return isNaN(start) || isNaN(end) || end <= start ? null : { start, end };
});

const plotW = W - PAD.l - PAD.r;
const xFor = (t: number) =>
  PAD.l + ((t - win.value!.start) / (win.value!.end - win.value!.start)) * plotW;
const rowCenter = (i: number) => PAD.t + i * ROW_H + ROW_H / 2;

const segments = computed(() => {
  if (!win.value) return [];
  return timed.value
    .map((s) => {
      const i = ROWS.findIndex((r) => r.key === s.stage);
      if (i < 0) return null;
      const x0 = Math.max(PAD.l, xFor(s.start));
      const x1 = Math.min(W - PAD.r, xFor(s.end));
      // 1px inset on both sides = surface gap between touching segments
      return {
        x: x0 + 0.5,
        w: Math.max(1.5, x1 - x0 - 1),
        y: rowCenter(i) - BAR / 2,
        stage: s.stage,
        start: s.start,
        end: s.end,
      };
    })
    .filter((s): s is NonNullable<typeof s> => s !== null);
});

const fmtTime = (t: number) =>
  new Date(t).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });

const xTicks = computed(() => {
  if (!win.value) return [];
  const mid = (win.value.start + win.value.end) / 2;
  return [
    { x: PAD.l, label: fmtTime(win.value.start), anchor: 'start' },
    { x: xFor(mid), label: fmtTime(mid), anchor: 'middle' },
    { x: W - PAD.r, label: fmtTime(win.value.end), anchor: 'end' },
  ];
});

const totals = computed(() => {
  const sums: Record<string, number> = {};
  for (const s of props.stages) {
    const key = norm(s.stage);
    if (key) sums[key] = (sums[key] || 0) + s.seconds / 60;
  }
  const total = Object.values(sums).reduce((a, b) => a + b, 0);
  if (!total) return [];
  return ROWS.filter((r) => sums[r.key]).map((r) => ({
    stage: r.key,
    label: r.label,
    minutes: sums[r.key],
    pct: (sums[r.key] / total) * 100,
  }));
});

function formatMin(min: number): string {
  const m = Math.round(min);
  return m >= 60 ? `${Math.floor(m / 60)}h ${m % 60}m` : `${m}m`;
}

const tipLeft = computed(() => {
  if (!hover.value) return '50%';
  const pct = (hover.value.x / W) * 100;
  return `${Math.min(Math.max(pct, 20), 80)}%`;
});

function onPointer(e: PointerEvent) {
  if (!svgEl.value || !win.value || !segments.value.length) return;
  const rect = svgEl.value.getBoundingClientRect();
  const xSvg = ((e.clientX - rect.left) / rect.width) * W;
  const t = win.value.start + ((xSvg - PAD.l) / plotW) * (win.value.end - win.value.start);
  const seg = segments.value.find((s) => t >= s.start && t <= s.end);
  if (!seg) {
    hover.value = null;
    return;
  }
  const row = ROWS.find((r) => r.key === seg.stage);
  const mins = Math.round((seg.end - seg.start) / 60000);
  hover.value = {
    x: Math.min(Math.max(xSvg, PAD.l), W - PAD.r),
    label: `${row?.label} ${fmtTime(seg.start)}-${fmtTime(seg.end)} · ${mins}m`,
  };
}
</script>

<style scoped>
/* Stage colors: CVD-validated palette (see theme/variables.css) */
.hyp__fill--wake { fill: var(--pulso-viz-wake); background: var(--pulso-viz-wake); }
.hyp__fill--rem { fill: var(--pulso-viz-rem); background: var(--pulso-viz-rem); }
.hyp__fill--light { fill: var(--pulso-viz-sleep-light); background: var(--pulso-viz-sleep-light); }
.hyp__fill--deep { fill: var(--pulso-viz-deep); background: var(--pulso-viz-deep); }

.hyp__plot {
  position: relative;
}
.hyp__svg {
  display: block;
  width: 100%;
  touch-action: pan-y;
}
.hyp__guide {
  stroke: var(--pulso-viz-grid);
  stroke-width: 1;
}
.hyp__row-label,
.hyp__tick {
  font-family: var(--ion-font-family);
  font-size: 8.5px;
  fill: var(--ion-color-medium);
}
.hyp__cross {
  stroke: var(--ion-color-medium);
  stroke-width: 1;
  opacity: 0.5;
}
.hyp__tip {
  position: absolute;
  top: -4px;
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

/* --- Composition bar + legend --- */
.hyp__composition {
  margin-top: 12px;
}
.hyp__bar {
  display: flex;
  gap: 2px;
  height: 10px;
}
.hyp__bar-seg {
  border-radius: 2px;
  min-width: 2px;
}
.hyp__bar-seg:first-child {
  border-top-left-radius: 5px;
  border-bottom-left-radius: 5px;
}
.hyp__bar-seg:last-child {
  border-top-right-radius: 5px;
  border-bottom-right-radius: 5px;
}
.hyp__legend {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 6px 14px;
  margin-top: 10px;
}
.hyp__legend-item {
  display: flex;
  align-items: baseline;
  gap: 6px;
  min-width: 0;
}
.hyp__swatch {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex: none;
  align-self: center;
}
.hyp__legend-label {
  font-family: var(--font-secondary);
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--ion-color-medium);
}
.hyp__legend-value {
  margin-left: auto;
  font-family: var(--ion-font-family);
  font-size: 11px;
  font-weight: 700;
  color: var(--ion-text-color);
  white-space: nowrap;
}
.hyp__legend-pct {
  font-family: var(--ion-font-family);
  font-size: 9px;
  color: var(--ion-color-medium);
  white-space: nowrap;
}
</style>
