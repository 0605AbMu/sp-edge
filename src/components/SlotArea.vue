<script setup lang="ts">
import { computed, ref } from 'vue'
import ParkingSlotGroup from './ParkingSlotGroup.vue'

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

const props = defineProps<{
  id: string
  config: {
    x: number
    y: number
    width: number
    height: number
    rotation?: number
    draggable?: boolean
  }
  slotCount: number
  slotWidth: number
  slotHeight: number
  slotAngle: number
  pixelsPerMeter: number
  textScale: number
  parkingWidth: number
  parkingHeight: number
}>()

const emit = defineEmits(['mouseenter', 'mouseleave', 'delete', 'rotate', 'edit', 'update-position'])

const isHovered = ref(false)

const handleDrag = (e: any) => {
  const node = e.currentTarget
  emit('update-position', {
    x: node.x(),
    y: node.y()
  })
}

const slantOffset = computed(() => {
  const sh = props.slotHeight * props.pixelsPerMeter
  const angleRad = (props.slotAngle * Math.PI) / 180
  return sh * Math.tan(angleRad)
})

const localBounds = computed(() => {
  const sw = props.slotWidth * props.pixelsPerMeter
  const totalWidth = props.slotCount * sw
  const sOffset = slantOffset.value
  
  return {
    x: Math.min(0, sOffset),
    y: 0,
    width: totalWidth + Math.abs(sOffset),
    height: props.slotHeight * props.pixelsPerMeter
  }
})

const generatedSlots = computed(() => {
  const slots: Slot[] = []
  const sw = props.slotWidth * props.pixelsPerMeter
  const sh = props.slotHeight * props.pixelsPerMeter
  const sOffset = slantOffset.value

  for (let i = 0; i < props.slotCount; i++) {
    const startX = i * sw
    
    // To'rtburchak o'rniga ixtiyoriy burchakli polygon (points) ishlatamiz
    // Nuqtalar tartibi: [x1, y1, x2, y2, x3, y3, x4, y4]
    const points = [
      startX, 0,                      // Yuqori chap
      startX + sw, 0,                 // Yuqori o'ng
      startX + sw + sOffset, sh,  // Pastki o'ng (surilgan)
      startX + sOffset, sh        // Pastki chap (surilgan)
    ]

    // Center in parking coords (meters)
    const cxPx = props.config.x + i * sw + (sw + sOffset) / 2
    const cyPx = props.config.y + sh / 2

    slots.push({
      id: `generated-${i}`,
      points,
      index: i + 1,
      cx: cxPx / props.pixelsPerMeter,
      cy: cyPx / props.pixelsPerMeter
    })
  }
  return slots
})

const bounds = computed(() => {
  const rotationRad = ((props.config.rotation || 0) * Math.PI) / 180
  const lb = localBounds.value

  // Barcha nuqtalarni yig'amiz (slantni hisobga olib, lekin rotation'siz)
  const basePoints = [
    { x: lb.x, y: lb.y },
    { x: lb.x + lb.width, y: lb.y },
    { x: lb.x + lb.width, y: lb.y + lb.height },
    { x: lb.x, y: lb.y + lb.height }
  ]

  // Endi bu nuqtalarni rotation bo'yicha aylantiramiz
  const rotatedPoints = basePoints.map(p => ({
    x: p.x * Math.cos(rotationRad) - p.y * Math.sin(rotationRad),
    y: p.x * Math.sin(rotationRad) + p.y * Math.cos(rotationRad)
  }))

  return {
    minX: Math.min(...rotatedPoints.map(p => p.x)),
    maxX: Math.max(...rotatedPoints.map(p => p.x)),
    minY: Math.min(...rotatedPoints.map(p => p.y)),
    maxY: Math.max(...rotatedPoints.map(p => p.y))
  }
})

const totalWidthWithSlant = computed(() => localBounds.value.width)

const mainGroupConfig = computed(() => ({
  id: props.id,
  x: props.config.x,
  y: props.config.y,
  draggable: props.config.draggable,
  name: 'slotArea',
  dragBoundFunc: function(this: any, pos: { x: number, y: number }) {
    const node = this
    const parent = node.getParent()
    const parentPos = parent.getAbsolutePosition()
    const stage = node.getStage()
    const scale = stage.scaleX()

    const { minX, maxX, minY, maxY } = bounds.value

    // Lokal koordinatalarni hisoblash
    let localX = (pos.x - parentPos.x) / scale
    let localY = (pos.y - parentPos.y) / scale

    // Chegaralash
    localX = Math.max(-minX, Math.min(props.parkingWidth - maxX, localX))
    localY = Math.max(-minY, Math.min(props.parkingHeight - maxY, localY))

    // Absolut koordinatalarga qaytarish
    return {
      x: localX * scale + parentPos.x,
      y: localY * scale + parentPos.y
    }
  }
}))

const rotatedGroupConfig = computed(() => ({
  rotation: props.config.rotation || 0
}))

const handleMouseEnter = (e: any) => {
  isHovered.value = true
  emit('mouseenter', e)
}

const handleMouseLeave = (e: any) => {
  isHovered.value = false
  emit('mouseleave', e)
}

const tooltipText = computed(() => {
  const x_m = (props.config.x / props.pixelsPerMeter).toFixed(2)
  const y_m = (props.config.y / props.pixelsPerMeter).toFixed(2)
  const w_m = props.slotWidth.toFixed(2)
  const h_m = props.slotHeight.toFixed(2)
  return `x: ${x_m}m  y: ${y_m}m\nw: ${w_m}m  h: ${h_m}m\nangle: ${props.slotAngle}°\nslots: ${props.slotCount}`
})

const handleEdit = (e: any) => {
  const stage = e.target.getStage()
  const pointer = stage.getPointerPosition()
  if (pointer) {
    emit('edit', {
      id: props.id,
      x: props.config.x,
      y: props.config.y,
      screenX: pointer.x,
      screenY: pointer.y
    })
  } else {
    emit('edit', { id: props.id, x: props.config.x, y: props.config.y })
  }
}
</script>

<template>
  <v-group 
    :config="mainGroupConfig"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @dragmove="handleDrag"
    @dragend="handleDrag"
  >
    <v-group :config="rotatedGroupConfig">
      <!-- Background for Slot Area -->
      <v-rect :config="{
        x: localBounds.x,
        y: localBounds.y,
        width: localBounds.width,
        height: localBounds.height,
        fill: '#f0f0f0',
        opacity: isHovered ? 0.3 : 0.15,
        stroke: isHovered ? 'blue' : 'gray',
        strokeWidth: 1,
        dash: isHovered ? [2, 2] : [5, 5],
        listening: true
      }" />

      <ParkingSlotGroup
        :config="{ x: 0, y: 0, width: localBounds.width, height: localBounds.height }"
        :slots="generatedSlots"
        :textScale="textScale"
        :rotation="props.config.rotation || 0"
      />
    </v-group>
    
    <v-label v-if="isHovered" :config="{ x: bounds.minX, y: bounds.minY - 5, scaleX: textScale, scaleY: textScale }">
      <v-tag :config="{ fill: 'black', opacity: 0.7, pointerDirection: 'down', pointerWidth: 8, pointerHeight: 8, lineJoin: 'round', cornerRadius: 4 }" />
      <v-text :config="{ text: tooltipText, fontSize: 12, fill: 'white', padding: 6, lineHeight: 1.5, listening: false }" />
    </v-label>

    <!-- Rotate, Edit va Delete tugmalari - har doim yuqori o'ng burchakda -->
    <v-group v-if="isHovered" :config="{ x: bounds.maxX, y: bounds.minY, scaleX: textScale, scaleY: textScale }">
      <!-- Rotate button -->
      <v-circle :config="{ x: -65, y: 15, radius: 12, fill: 'white', stroke: 'blue' }" @click="emit('rotate')" />
      <v-text :config="{ text: '↻', x: -72, y: 7, fontSize: 16, fill: 'blue', listening: false }" />

      <!-- Edit button -->
      <v-circle :config="{ x: -40, y: 15, radius: 12, fill: 'white', stroke: 'green' }" @click="handleEdit" />
      <v-text :config="{ text: '✎', x: -46, y: 6, fontSize: 14, fill: 'green', listening: false }" />

      <!-- Delete button -->
      <v-circle :config="{ x: -15, y: 15, radius: 12, fill: 'white', stroke: 'red' }" @click="emit('delete')" />
      <v-text :config="{ text: '×', x: -22, y: 5, fontSize: 18, fill: 'red', listening: false }" />
    </v-group>
  </v-group>
</template>
