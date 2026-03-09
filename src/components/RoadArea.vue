<script setup lang="ts">
import { computed, ref } from 'vue'

const props = defineProps<{
  id: string
  x: number
  y: number
  width: number  // meters
  height: number // meters
  rotation: number
  name: string
  pixelsPerMeter: number
  textScale: number
  isSelected: boolean
  isReadonly?: boolean
}>()

const emit = defineEmits(['select', 'update-position', 'delete', 'dragmove'])

const isHovered = ref(false)

const wPx = computed(() => props.width * props.pixelsPerMeter)
const hPx = computed(() => props.height * props.pixelsPerMeter)

const groupConfig = computed(() => ({
  x: props.x,
  y: props.y,
  rotation: props.rotation,
  draggable: !props.isReadonly,
  listening: true,
  id: props.id
}))

const rectConfig = computed(() => ({
  x: 0,
  y: 0,
  width: wPx.value,
  height: hPx.value,
  fill: '#455a64',
  stroke: props.isSelected ? '#f1c40f' : (isHovered.value ? '#3498db' : '#37474f'),
  strokeWidth: props.isSelected ? 3 : 2,
  opacity: 0.9,
  cornerRadius: 2
}))

const dashedLineConfig = computed(() => ({
  points: [5, hPx.value / 2, wPx.value - 5, hPx.value / 2],
  stroke: 'rgba(255, 255, 255, 0.5)',
  strokeWidth: 2,
  dash: [15, 10],
  lineCap: 'round'
}))

const labelConfig = computed(() => ({
  x: wPx.value / 2,
  y: hPx.value / 2,
  scaleX: props.textScale,
  scaleY: props.textScale,
  rotation: -props.rotation,
  listening: false,
  offsetY: 8
}))

const textConfig = computed(() => ({
  text: props.name || 'Yo\'lak',
  fontSize: 12,
  fontStyle: 'bold',
  fill: 'rgba(255, 255, 255, 0.8)',
  align: 'center',
  verticalAlign: 'middle',
  width: 120,
  offsetX: 60
}))

const handleDragMove = (e: any) => {
  emit('update-position', {
    id: props.id,
    x: e.target.x(),
    y: e.target.y()
  })
  emit('dragmove', e)
}

const handleClick = (e: any) => {
  e.cancelBubble = true
  emit('select', props.id)
}
</script>

<template>
  <v-group
    :config="groupConfig"
    @dragmove="handleDragMove"
    @click="handleClick"
    @mouseenter="isHovered = true"
    @mouseleave="isHovered = false"
  >
    <!-- Yo'l asosi -->
    <v-rect :config="rectConfig" />
    
    <!-- Markaziy chiziq -->
    <v-line :config="dashedLineConfig" />
    
    <!-- Yo'l nomi -->
    <v-label :config="labelConfig">
      <v-text :config="textConfig" />
    </v-label>
  </v-group>
</template>
