<template>
  <div class="zones">
    <div v-for="z in rows" :key="z.name" class="zones__row">
      <div class="zones__head">
        <span class="zones__name">{{ z.name }}</span>
        <span class="zones__range">{{ z.min }}-{{ z.max }} bpm</span>
      </div>
      <div class="zones__bar-row">
        <div class="zones__bar" :style="{ width: `calc((100% - 52px) * ${(z.pct / 100).toFixed(4)})`, background: z.color }" />
        <span class="zones__value">{{ formatMin(z.minutes) }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue';

interface HeartRateZone {
  name: string;
  min: number;
  max: number;
  minutes: number;
  caloriesOut: number;
}

const props = defineProps<{ zones: HeartRateZone[] }>();

// Highest intensity first; ordinal single-hue ramp maps intensity to color
const ORDER = ['Peak', 'Cardio', 'Fat Burn', 'Out of Range'];
const COLOR_VARS: Record<string, string> = {
  'Out of Range': 'var(--pulso-viz-zone-1)',
  'Fat Burn': 'var(--pulso-viz-zone-2)',
  'Cardio': 'var(--pulso-viz-zone-3)',
  'Peak': 'var(--pulso-viz-zone-4)',
};

const rows = computed(() => {
  const sorted = [...props.zones].sort((a, b) => {
    const ia = ORDER.indexOf(a.name);
    const ib = ORDER.indexOf(b.name);
    return (ia < 0 ? 99 : ia) - (ib < 0 ? 99 : ib);
  });
  const max = Math.max(1, ...sorted.map((z) => z.minutes));
  return sorted.map((z) => ({
    ...z,
    pct: (z.minutes / max) * 100,
    color: COLOR_VARS[z.name] || 'var(--pulso-viz-zone-2)',
  }));
});

function formatMin(min: number): string {
  const m = Math.round(min);
  return m >= 60 ? `${Math.floor(m / 60)}h ${m % 60}m` : `${m}m`;
}
</script>

<style scoped>
.zones {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.zones__head {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-bottom: 3px;
}
.zones__name {
  font-family: var(--font-secondary);
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: var(--ion-color-medium);
}
.zones__range {
  font-family: var(--ion-font-family);
  font-size: 9px;
  color: var(--ion-color-medium);
  opacity: 0.8;
}
.zones__bar-row {
  display: flex;
  align-items: center;
  gap: 8px;
}
.zones__bar {
  height: 12px;
  min-width: 2px;
  border-radius: 0 4px 4px 0;
  flex: none;
}
.zones__value {
  font-family: var(--ion-font-family);
  font-size: 11px;
  font-weight: 700;
  color: var(--ion-text-color);
  white-space: nowrap;
}
</style>
