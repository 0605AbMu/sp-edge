<script setup lang="ts">
import { computed, ref } from 'vue'
import ParkingSlot from './ParkingSlot.vue'

interface Slot {
  id: string
  x?: number
  y?: number
  width?: number
  height?: number
  points?: number[]
  index?: number
  cx?: number
  cy?: number
}

interface HoveredInfo {
  shapeCenter: { x: number; y: number }
  index: number
  cx: number
  cy: number
}

const props = defineProps<{
  config: {
    x: number
    y: number
    width: number
    height: number
  }
  slots: Slot[]
  name?: string
  textScale: number
  rotation?: number
}>()

const hoveredSlot = ref<HoveredInfo | null>(null)

const tooltipText = computed(() => {
  if (!hoveredSlot.value) return ''
  const { index, cx, cy } = hoveredSlot.value
  return `#${index}\nx: ${cx.toFixed(2)}m\ny: ${cy.toFixed(2)}m`
})
</script>

<template>
  <v-group :config="{ x: config.x, y: config.y }">
    <v-rect :config="{
      width: config.width,
      height: config.height,
      stroke: 'gray',
      strokeWidth: 1,
      dash: [5, 5],
      listening: false
    }" />

    <ParkingSlot
      v-for="slot in slots"
      :key="slot.id"
      :config="{ ...slot, fill: '#a5d6a7' }"
      :index="slot.index"
      :cx="slot.cx"
      :cy="slot.cy"
      :textScale="textScale"
      :counterRotation="-(rotation || 0)"
      @hovered="hoveredSlot = $event"
      @unhovered="hoveredSlot = null"
    />

    <!-- Tooltip: barcha slotlardan KEYIN, teskari aylantirilgan holda -->
    <v-group
      v-if="hoveredSlot"
      :config="{
        x: hoveredSlot.shapeCenter.x,
        y: hoveredSlot.shapeCenter.y,
        rotation: -(rotation || 0),
        listening: false
      }"
    >
      <v-label :config="{ x: 0, y: 0, scaleX: textScale, scaleY: textScale, listening: false }">
        <v-tag :config="{ fill: '#1a1a1a', opacity: 0.92, cornerRadius: 4 }" />
        <v-text :config="{ text: tooltipText, fontSize: 14, fill: 'white', padding: 6, lineHeight: 1.6, listening: false }" />
      </v-label>
    </v-group>

    <v-text v-if="name" :config="{
      text: name,
      x: 5,
      y: -15,
      fontSize: 12,
      fill: 'gray',
      scaleX: textScale,
      scaleY: textScale
    }" />
  </v-group>
</template>
