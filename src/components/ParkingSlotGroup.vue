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

const emit = defineEmits([])

const hoveredSlot = ref<HoveredInfo | null>(null)

const handleMouseEnterSlot = (event: HoveredInfo) => {
  hoveredSlot.value = event
}

const handleMouseLeaveSlot = (event: HoveredInfo) => {
  hoveredSlot.value = null
}

</script>

<template>
  <v-group :config="{ x: config.x, y: config.y }">
    <ParkingSlot
      v-for="slot in slots"
      :key="slot.id"
      :config="{ ...slot, fill: 'transparent', stroke: '#ffffff', strokeWidth: 2 }"
      :index="slot.index"
      :cx="slot.cx"
      :cy="slot.cy"
      :textScale="1"
      :counterRotation="-(rotation || 0)"
      @hovered="handleMouseEnterSlot"
      @unhovered="handleMouseLeaveSlot($event)"
    />

    <v-text v-if="name" :config="{
      text: name,
      x: 5,
      y: -15,
      fontSize: 12,
      fill: 'gray'
    }" />
  </v-group>
</template>
