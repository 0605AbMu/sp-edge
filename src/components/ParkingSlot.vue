<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{
  config: {
    points?: number[]
    x?: number
    y?: number
    width?: number
    height?: number
    fill?: string
    opacity?: number
    stroke?: string
    strokeWidth?: number
  }
  isSelected?: boolean
  index?: number
  cx?: number
  cy?: number
  textScale?: number
  counterRotation?: number
}>()

const emit = defineEmits<{
  hovered: [info: { shapeCenter: { x: number; y: number }; index: number; cx: number; cy: number }]
  unhovered: []
}>()

const slotConfig = computed(() => ({
  ...props.config,
  stroke: props.isSelected ? '#00bfff' : (props.config.stroke || '#ffffff'),
  strokeWidth: props.isSelected ? 3 : (props.config.strokeWidth || 2),
  fill: props.config.fill || 'transparent',
  opacity: props.config.opacity || 1,
  closed: props.config.points ? true : undefined
}))

const shapeCenter = computed(() => {
  if (props.config.points && props.config.points.length >= 8) {
    const pts = props.config.points
    return {
      x: (pts[0] + pts[2] + pts[4] + pts[6]) / 4,
      y: (pts[1] + pts[3] + pts[5] + pts[7]) / 4
    }
  }
  return {
    x: (props.config.x || 0) + (props.config.width || 0) / 2,
    y: (props.config.y || 0) + (props.config.height || 0) / 2
  }
})

const handleMouseEnter = () => {
  if (props.index !== undefined && props.cx !== undefined && props.cy !== undefined) {
    emit('hovered', { shapeCenter: shapeCenter.value, index: props.index, cx: props.cx, cy: props.cy })
  }
}
</script>

<template>
  <v-group>
    <v-line v-if="config.points" :config="slotConfig" @mouseenter="handleMouseEnter" @mouseleave="emit('unhovered')" />
    <v-rect v-else :config="slotConfig" @mouseenter="handleMouseEnter" @mouseleave="emit('unhovered')" />

    <!-- Tartib raqami: shape markazi atrofida teskari aylanadi — foydalanuvchiga to'g'ri ko'rinadi -->
    <v-group
      v-if="index !== undefined"
      :config="{ x: shapeCenter.x, y: shapeCenter.y, rotation: counterRotation || 0, listening: false }"
    >
      <v-text :config="{
        x: 0, y: 0,
        width: 40, height: 30,
        offsetX: 20, offsetY: 15,
        text: String(index),
        align: 'center',
        verticalAlign: 'middle',
        fontSize: 22,
        fill: '#ffffff',
        fontStyle: 'bold',
        listening: false
      }" />
    </v-group>
  </v-group>
</template>
