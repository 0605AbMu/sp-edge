<script setup lang="ts">
import { computed, ref } from 'vue'

const props = defineProps<{
  id: string
  side: 'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT'
  x: number
  y: number
  width: number  // meters
  type: 'ENTRANCE' | 'EXIT' | 'BOTH'
  isOpen?: boolean
  rotation?: number
  pixelsPerMeter: number
  textScale: number
  parkingX: number
  parkingY: number
  parkingWidth: number
  parkingHeight: number
  isSelected: boolean
  isReadonly?: boolean
}>()

const emit = defineEmits(['select', 'delete', 'update-position', 'update-gate', 'dragmove'])

const isHovered = ref(false)

const toggleGate = () => {
  emit('update-gate', { id: props.id, isOpen: !props.isOpen })
}

const halfWPx = computed(() => props.width * props.pixelsPerMeter)

const PILLAR_W = 10
const PILLAR_H = 30

const gateRotation = computed(() => {
  let baseRotation = 0
  switch (props.side) {
    case 'TOP': baseRotation = 0; break
    case 'BOTTOM': baseRotation = 180; break
    case 'LEFT': baseRotation = 90; break
    case 'RIGHT': baseRotation = 270; break
  }
  return baseRotation + (props.rotation || 0)
})

const typeColor = computed(() => {
  switch (props.type) {
    case 'ENTRANCE': return '#4caf50'
    case 'EXIT': return '#f44336'
    case 'BOTH': return '#ff9800'
    default: return '#4caf50'
  }
})

const typeLabel = computed(() => {
  switch (props.type) {
    case 'ENTRANCE': return 'Kirish'
    case 'EXIT': return 'Chiqish'
    case 'BOTH': return 'Kirish/Chiqish'
    default: return ''
  }
})

const dragBoundFunc = function (this: any, pos: { x: number; y: number }) {
  const stage = this.getStage()
  const scale = stage.scaleX()
  
  // Convert global pos to parking-relative pixels
  const parentPos = this.getParent().getAbsolutePosition()
  
  let lx = (pos.x - parentPos.x) / scale - props.parkingX
  let ly = (pos.y - parentPos.y) / scale - props.parkingY

  // IMPORTANT: Gate group has offsetX/offsetY which shifts its (0,0) point.
  // When Konva calculates position during drag, it accounts for these offsets.
  // Since we are overriding the position via dragBoundFunc, we MUST add back 
  // the current offsets to get the correct logical (x,y) that corresponds to 
  // the visual top-left corner we want to constrain.
  lx += gateOffsets.value.x
  ly += gateOffsets.value.y

  // Constrain to parking area (allow origin to be at 0,0)
  // No more constraints - allow moving everywhere
  // lx = Math.max(0, Math.min(props.parkingWidth, lx))
  // ly = Math.max(0, Math.min(props.parkingHeight, ly))

  // Return stage-relative pixels
  return {
    x: (lx - gateOffsets.value.x + props.parkingX) * scale + parentPos.x,
    y: (ly - gateOffsets.value.y + props.parkingY) * scale + parentPos.y
  }
}

const handleDragEnd = (e: any) => {
  // We need to pass the position that App.vue expects.
  // App.vue expects gate.x + configRect.x
  // e.target.x() is relative to parent, which is the same as configRect's parent.
  emit('update-position', { id: props.id, x: e.target.x(), y: e.target.y() })
}

const handleDragMove = (e: any) => {
  emit('dragmove', e)
}

const gateOffsets = computed(() => {
  const rad = (gateRotation.value * Math.PI) / 180
  
  // Element width and height in stage pixels
  const w = halfWPx.value
  const h = PILLAR_H
  
  // Local corners of the unrotated elements [0, 0, w, h]
  const corners = [
    { x: 0, y: 0 },
    { x: w, y: 0 },
    { x: w, y: h },
    { x: 0, y: h }
  ]
  
  // Rotate corners around local (0,0) to find visual bounding box
  const rotatedCorners = corners.map(p => ({
    x: p.x * Math.cos(rad) - p.y * Math.sin(rad),
    y: p.x * Math.sin(rad) + p.y * Math.cos(rad)
  }))
  
  // Find top-left of the visual bounding box in rotated space
  const minX = Math.min(...rotatedCorners.map(p => p.x))
  const minY = Math.min(...rotatedCorners.map(p => p.y))
  
  // To keep (x,y) at the visual top-left corner after rotation,
  // we use the inverse rotation matrix to find which local point (ox, oy) 
  // maps to (minX, minY) in the stage coordinate system.
  const cos = Math.cos(rad)
  const sin = Math.sin(rad)
  
  // Inverse rotation: x_local = x_rot * cos + y_rot * sin
  //                  y_local = -x_rot * sin + y_rot * cos
  return {
    x: minX * cos + minY * sin,
    y: -minX * sin + minY * cos
  }
})
</script>

<template>
  <v-group
    :config="{
      x,
      y,
      rotation: gateRotation,
      offsetX: gateOffsets.x,
      offsetY: gateOffsets.y,
      draggable: !isReadonly,
      name: 'gate',
      id
    }"
    @click="emit('select', id)"
    @mouseenter="isHovered = true"
    @mouseleave="isHovered = false"
    @dragmove="handleDragMove"
    @dragend="handleDragEnd"
  >
    <!-- Barrier Components Group -->
    <v-group :config="{ 
      scaleX: 1, 
      scaleY: 1,
      offsetX: 0,
      offsetY: 0
    }">
      <!-- Left pillar -->
      <v-rect :config="{
        x: 0,
        y: 0,
        width: PILLAR_W,
        height: PILLAR_H,
        fill: typeColor,
        cornerRadius: [3, 0, 0, 3],
        shadowColor: 'rgba(0,0,0,0.6)',
        shadowBlur: 5,
        shadowOffset: { x: 1, y: 2 },
        name: 'gate'
      }" />

      <!-- Right pillar -->
      <v-rect :config="{
        x: halfWPx - PILLAR_W,
        y: 0,
        width: PILLAR_W,
        height: PILLAR_H,
        fill: typeColor,
        cornerRadius: [0, 3, 3, 0],
        shadowColor: 'rgba(0,0,0,0.6)',
        shadowBlur: 5,
        shadowOffset: { x: 1, y: 2 },
        name: 'gate'
      }" />

      <!-- ═══ PARKING BARRIER (boom gate) ═══ -->
      <!-- Barrier housing box (motor unit) — on left pillar inner side -->
      <v-group :config="{ x: PILLAR_W, y: PILLAR_H / 2 }">
        <!-- Asosiy korpus -->
        <v-rect :config="{
          x: -8, y: -15,
          width: 16, height: 30,
          fill: '#2c3e50',
          stroke: isSelected ? '#f5c518' : '#34495e',
          strokeWidth: 2,
          cornerRadius: 4,
          shadowColor: 'black',
          shadowBlur: 10,
          shadowOpacity: 0.3,
          shadowOffset: { x: 2, y: 2 }
        }" />
        <!-- Yuqori qopqoq -->
        <v-rect :config="{
          x: -9, y: -16,
          width: 18, height: 6,
          fill: '#1a252f',
          cornerRadius: 2
        }" />
        <!-- Status LED chirog'i -->
        <v-circle :config="{
          x: 0, y: -8,
          radius: 3,
          fill: isOpen ? '#2ecc71' : '#e74c3c',
          shadowColor: isOpen ? '#2ecc71' : '#e74c3c',
          shadowBlur: 8,
          shadowOpacity: 0.8
        }" />
        
        <!-- Boom arm (Shlagbaum to'sig'i) -->
        <v-group :config="{ 
          x: 0, y: -4,
          rotation: isOpen ? -90 : 0
        }" @click.stop="toggleGate">
          <!-- To'siq tayog'i (Sariq-qora chiziqli) -->
          <v-rect :config="{
            x: 0, y: -3,
            width: halfWPx - (PILLAR_W * 2),
            height: 6,
            fill: '#f1c40f',
            cornerRadius: 3,
            stroke: '#f39c12',
            strokeWidth: 1
          }" />
          <!-- Qora chiziqlar -->
          <v-line v-for="seg in Math.floor((halfWPx - (PILLAR_W * 2)) / 20)" :key="'stripe-' + seg" :config="{
            points: [seg * 20 - 10, -3, seg * 20, 3],
            stroke: '#2c3e50',
            strokeWidth: 8,
            lineCap: 'butt'
          }" />
          <!-- To'siq uchi -->
          <v-circle :config="{
            x: halfWPx - (PILLAR_W * 2), y: 0,
            radius: 3,
            fill: '#e74c3c'
          }" />
        </v-group>
      </v-group>

      <!-- Selection / hover ring -->
      <v-rect v-if="isSelected || isHovered" :config="{
        x: -2,
        y: -2,
        width: halfWPx + 4,
        height: PILLAR_H + 4,
        fill: 'transparent',
        stroke: typeColor,
        strokeWidth: isSelected ? 2 : 1,
        dash: [5, 3],
        cornerRadius: 4,
        listening: false,
        opacity: isSelected ? 1 : 0.5
      }" />
    </v-group>

    <!-- Boundary dashed line between pillars -->
    <v-line :config="{
      points: [PILLAR_W, 0, halfWPx - PILLAR_W, 0],
      stroke: typeColor,
      strokeWidth: 2,
      dash: [6, 4],
      opacity: 0.25,
      listening: false
    }" />

    <!-- Selection / hover ring (Moved inside constant-size group above) -->

    <!-- ═══ INPUT ARROW (Floating Arrow) ═══ -->
    <v-group :config="{
      x: halfWPx / 2,
      y: -40,
      scaleX: textScale,
      scaleY: textScale,
      listening: false
    }">
      <v-arrow :config="{
        points: [0, -15, 0, 15],
        pointerLength: 10,
        pointerWidth: 10,
        fill: typeColor,
        stroke: typeColor,
        strokeWidth: 4,
        shadowColor: 'black',
        shadowBlur: 5,
        shadowOpacity: 0.3,
        lineCap: 'round',
        lineJoin: 'round'
      }" />
    </v-group>
  </v-group>
</template>
