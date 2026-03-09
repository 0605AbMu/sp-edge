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
  isReadonly?: boolean
}>()

const emit = defineEmits(['edit', 'update-position', 'dragmove', 'hover-change'])

const isHovered = ref(false)

const handleMouseEnter = (e: any) => {
  isHovered.value = true
  emit('hover-change', {
    id: props.id,
    isHovered: true,
    x: props.config.x,
    y: props.config.y,
    width: localBounds.value.width,
    height: localBounds.value.height,
    slotCount: props.slotCount,
    pixelsPerMeter: props.pixelsPerMeter,
    rotation: props.config.rotation || 0
  })
}

const handleMouseLeave = (e: any) => {
  isHovered.value = false
  emit('hover-change', {
    id: props.id,
    isHovered: false
  })
}

const handleDrag = (e: any) => {
  const node = e.currentTarget
  emit('update-position', {
    x: node.x(),
    y: node.y()
  })
  emit('dragmove', e)
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
  
  const minX = Math.min(0, sOffset)
  const maxX = Math.max(totalWidth, totalWidth + sOffset)
  
  return {
    x: 0,
    y: 0,
    width: maxX - minX,
    height: props.slotHeight * props.pixelsPerMeter
  }
})

const generatedSlots = computed(() => {
  const slots: Slot[] = []
  const sw = props.slotWidth * props.pixelsPerMeter
  const sh = props.slotHeight * props.pixelsPerMeter
  const sOffset = slantOffset.value
  const offsetX = -Math.min(0, sOffset) // Agar qiyalik chapga bo'lsa, slotlarni o'ngga suramiz
  
  for (let i = 0; i < props.slotCount; i++) {
    const startX = i * sw + offsetX
    
    // To'rtburchak o'rniga ixtiyoriy burchakli polygon (points) ishlatamiz
    // Nuqtalar tartibi: [x1, y1, x2, y2, x3, y3, x4, y4]
    const points = [
      startX, 0,                      // Yuqori chap
      startX + sw, 0,                 // Yuqori o'ng
      startX + sw + sOffset, sh,      // Pastki o'ng (surilgan)
      startX + sOffset, sh            // Pastki chap (surilgan)
    ]

    // Center in group local coords (meters)
    const cxPx = startX + (sw + sOffset) / 2
    const cyPx = sh / 2

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
    { x: 0, y: lb.y },
    { x: lb.width, y: lb.y },
    { x: lb.width, y: lb.y + lb.height },
    { x: 0, y: lb.y + lb.height }
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
  draggable: !props.isReadonly && props.config.draggable,
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

const handleClick = (e: any) => {
  if (props.isReadonly) return
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
    @click="handleClick"
  >
    <v-group :config="rotatedGroupConfig">
      <!-- Background for Slot Area -->
      <v-rect :config="{
        x: 0,
        y: 0,
        width: localBounds.width,
        height: localBounds.height,
        fill: '#1a1a1a',
        opacity: 0.5,
        stroke: isHovered && !isReadonly ? '#00bfff' : '#ffffff',
        strokeWidth: isHovered && !isReadonly ? 2 : 1,
        listening: true
      }" />

      <ParkingSlotGroup
        :config="{ x: 0, y: 0, width: localBounds.width, height: localBounds.height }"
        :slots="generatedSlots"
        :textScale="textScale"
        :rotation="props.config.rotation || 0"
      />
    </v-group>
  </v-group>
</template>
