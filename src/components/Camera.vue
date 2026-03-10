<script setup lang="ts">
import { computed, ref } from 'vue'

const props = defineProps<{
  id: string
  x: number
  y: number
  corners: number[]
  textScale: number
  parkingWidth: number
  parkingHeight: number
  pixelsPerMeter: number
  isSelected: boolean
  isReadonly?: boolean
}>()

const emit = defineEmits(['select', 'update-body', 'update-corner', 'dragmove'])

const isBodyHovered = ref(false)
const hoveredCorner = ref<number | null>(null)
const isDragging = ref(false)
const liveDragOffset = ref({ dx: 0, dy: 0 })
const dragSnapshot = ref<{ bodyX: number; bodyY: number; corners: number[] } | null>(null)

const handleDragStart = () => {
  dragSnapshot.value = { bodyX: props.x, bodyY: props.y, corners: [...props.corners] }
  isDragging.value = true
  liveDragOffset.value = { dx: 0, dy: 0 }
}

const handleDragMove = (e: any) => {
  if (!dragSnapshot.value) return
  const dx = e.currentTarget.x() - dragSnapshot.value.bodyX
  const dy = e.currentTarget.y() - dragSnapshot.value.bodyY
  liveDragOffset.value = { dx, dy }
  // emit('dragmove', e) // removed to fix coverage area update during drag
}

const handleDragEnd = (e: any) => {
  if (!dragSnapshot.value) return
  const newX = e.currentTarget.x()
  const newY = e.currentTarget.y()
  const dx = newX - dragSnapshot.value.bodyX
  const dy = newY - dragSnapshot.value.bodyY
  const newCorners = dragSnapshot.value.corners.map((v, i) => i % 2 === 0 ? v + dx : v + dy)
  emit('update-body', { id: props.id, x: newX, y: newY, corners: newCorners })
  dragSnapshot.value = null
  isDragging.value = false
  liveDragOffset.value = { dx: 0, dy: 0 }
}

// Drag paytida corners live siljiydi
const displayCorners = computed(() => {
  const { dx, dy } = liveDragOffset.value
  if (!isDragging.value || (dx === 0 && dy === 0)) return props.corners
  return props.corners.map((v, i) => i % 2 === 0 ? v + dx : v + dy)
})

// Drag paytida camera markaziy pozitsiyasi
const displayX = computed(() => isDragging.value ? props.x + liveDragOffset.value.dx : props.x)
const displayY = computed(() => isDragging.value ? props.y + liveDragOffset.value.dy : props.y)

const centroid = computed(() => {
  const c = displayCorners.value
  return {
    x: (c[0] + c[2] + c[4] + c[6]) / 4,
    y: (c[1] + c[3] + c[5] + c[7]) / 4
  }
})

const facingAngleDeg = computed(() =>
  Math.atan2(centroid.value.y - displayY.value, centroid.value.x - displayX.value) * 180 / Math.PI
)

// Shoelace formula — kuzatuv maydoni (m²)
const coverageAreaM2 = computed(() => {
  const c = displayCorners.value
  const p = props.pixelsPerMeter
  const areaPx2 = 0.5 * Math.abs(
    c[0] * (c[3] - c[7]) +
    c[2] * (c[5] - c[1]) +
    c[4] * (c[7] - c[3]) +
    c[6] * (c[1] - c[5])
  )
  return (areaPx2 / (p * p)).toFixed(1)
})


const bodyColor = computed(() =>
  isBodyHovered.value ? '#00acc1' : (props.isSelected ? '#0097a7' : '#00bcd4')
)

const iconDragBound = function (this: any, pos: { x: number; y: number }) {
  const parentPos = this.getParent().getAbsolutePosition()
  const scale = this.getStage().scaleX()
  let lx = (pos.x - parentPos.x) / scale
  let ly = (pos.y - parentPos.y) / scale
  lx = Math.max(0, Math.min(props.parkingWidth, lx))
  ly = Math.max(0, Math.min(props.parkingHeight, ly))
  return { x: lx * scale + parentPos.x, y: ly * scale + parentPos.y }
}

const cornerDragBound = function (this: any, pos: { x: number; y: number }) {
  const parentPos = this.getParent().getParent().getAbsolutePosition()
  const scale = this.getStage().scaleX()
  let lx = (pos.x - parentPos.x) / scale
  let ly = (pos.y - parentPos.y) / scale
  lx = Math.max(0, Math.min(props.parkingWidth, lx))
  ly = Math.max(0, Math.min(props.parkingHeight, ly))
  return { x: lx * scale + parentPos.x, y: ly * scale + parentPos.y }
}

const handleCornerDragEnd = (e: any, index: number) => {
  emit('update-corner', { id: props.id, index, x: e.target.x(), y: e.target.y() })
}
</script>

<template>
  <v-group :config="{ name: 'camera', id }">

    <!-- ═══ COVERAGE: Har doim ko'rinadi (z-index pastroqda bo'lishi uchun oldinroq chiziladi) ═══ -->
    <!-- Kuzatuv maydoni to'ldirish -->
    <v-line :config="{
      points: displayCorners,
      fill: isSelected || isDragging ? (isReadonly ? 'rgba(0,255,100,0.3)' : 'rgba(0,220,255,0.25)') : (isReadonly ? 'rgba(0,255,100,0.15)' : 'rgba(0,220,255,0.12)'),
      stroke: isSelected || isDragging ? (isReadonly ? 'rgba(0,255,100,0.9)' : 'rgba(0,220,255,0.8)') : (isReadonly ? 'rgba(0,255,100,0.5)' : 'rgba(0,220,255,0.4)'),
      strokeWidth: isSelected ? 2 : 1.5,
      closed: true,
      listening: false
    }" />

    <!-- Kameradan burchaklargacha nurlar -->
    <v-line v-for="i in 4" :key="'ray-' + i" :config="{
      points: [displayX, displayY, displayCorners[(i - 1) * 2], displayCorners[(i - 1) * 2 + 1]],
      stroke: isSelected || isDragging ? (isReadonly ? 'rgba(0,255,100,0.4)' : 'rgba(0,220,255,0.3)') : (isReadonly ? 'rgba(0,255,100,0.25)' : 'rgba(0,220,255,0.2)'),
      strokeWidth: 0.8,
      dash: [6, 4],
      listening: false
    }" />

    <!-- ═══ BOSHQA ELEMENTLAR ═══ -->
    <!-- Maydon hajmi — har doim markazda ko'rinadi -->
    <v-group :config="{ x: centroid.x, y: centroid.y, listening: false }">
      <v-label :config="{ scaleX: textScale, scaleY: textScale, listening: false }">
        <v-tag :config="{
          fill: 'rgba(0,0,0,0.65)',
          cornerRadius: 4,
          stroke: isReadonly ? 'rgba(0,255,100,0.6)' : 'rgba(0,220,255,0.5)',
          strokeWidth: 1 / textScale
        }" />
        <v-text :config="{
          text: coverageAreaM2 + ' m²',
          fontSize: 13,
          fill: isReadonly ? '#00ff64' : '#00dfff',
          fontStyle: 'bold',
          padding: 5,
          listening: false,
          align: 'center',
          verticalAlign: 'middle'
        }" />
      </v-label>
    </v-group>

    <template v-if="isSelected || isDragging">
      <!-- Burchak tutqichlari (faqat isSelected da) -->
      <template v-if="isSelected && !isDragging && !isReadonly">
        <template v-for="i in 4" :key="'c-' + i">
          <v-circle :config="{
            x: displayCorners[(i - 1) * 2],
            y: displayCorners[(i - 1) * 2 + 1],
            radius: 8,
            fill: hoveredCorner === i - 1 ? '#ff9800' : '#fff',
            stroke: '#ff9800',
            strokeWidth: 3,
            draggable: true,
            scaleX: textScale,
            scaleY: textScale,
            dragBoundFunc: cornerDragBound,
            name: 'cameraCorner',
            shadowColor: 'black',
            shadowBlur: 4,
            shadowOpacity: 0.3,
            offsetX: 0,
            offsetY: 0
          }"
            @mouseenter="hoveredCorner = i - 1"
            @mouseleave="hoveredCorner = null"
            @dragend="(e: any) => handleCornerDragEnd(e, i - 1)"
          />
        </template>
      </template>

    </template>

    <!-- ═══ OUTDOOR BULLET CAMERA IKONASI ═══ -->
    <v-group
      :config="{
        x, y,
        scaleX: textScale,
        scaleY: textScale,
        draggable: !isReadonly,
        dragBoundFunc: iconDragBound,
        name: 'cameraBody',
        offsetX: 0,
        offsetY: 0
      }"
      @mouseenter="isBodyHovered = !isReadonly"
      @mouseleave="isBodyHovered = false"
      @click="emit('select', id)"
      @dragstart="handleDragStart"
      @dragmove="handleDragMove"
      @dragend="handleDragEnd"
    >
      <!-- Tanlov halqasi -->
      <v-rect v-if="isSelected" :config="{
        x: -14, y: -14, width: 28, height: 28,
        fill: 'transparent',
        stroke: '#00dfff',
        strokeWidth: 1.5 / textScale,
        dash: [4 / textScale, 3 / textScale],
        cornerRadius: 6,
        listening: false
      }" />

      <!-- Devor kreplaj plitasi (qorong'i kvadrat, aylanmaydi) -->
      <v-rect :config="{
        x: -12, y: -12, width: 24, height: 24,
        fill: '#1e2535',
        stroke: isSelected ? '#00dfff' : (isBodyHovered ? '#4a90d9' : '#2e3d50'),
        strokeWidth: 1.5 / textScale,
        cornerRadius: 5,
        shadowColor: 'rgba(0,0,0,0.8)',
        shadowBlur: 6,
        shadowOffset: { x: 1, y: 2 }
      }" />
      <!-- L-bracket ichki qirqim -->
      <v-rect :config="{
        x: 4, y: -8, width: 14, height: 16,
        fill: '#162030',
        cornerRadius: [0, 3, 3, 0],
        listening: false
      }" />

      <!-- Kamera yo'nalishida aylanadigan qism (+X = linza = coverage tomonga) -->
      <v-group :config="{ rotation: facingAngleDeg }">

        <!-- Bracket arm -->
        <v-rect :config="{
          x: 10, y: -5, width: 16, height: 10,
          fill: '#2e3d50',
          cornerRadius: 2
        }" />

        <!-- Arm-body ulanish nuqtasi -->
        <v-rect :config="{
          x: 23, y: -12, width: 6, height: 24,
          fill: '#253040',
          cornerRadius: 1
        }" />

        <!-- === KAMERA ASOSIY KORPUSI (oq/kulrang, engil rang) === -->
        <!-- Pastki soya (3D ko'rinish) -->
        <v-rect :config="{
          x: 27, y: 5, width: 32, height: 7,
          fill: '#8fa8bc',
          cornerRadius: [0, 0, 3, 0],
          listening: false
        }" />

        <!-- Asosiy kuzov (oq-kulrang) -->
        <v-rect :config="{
          x: 27, y: -12, width: 32, height: 17,
          fill: '#d0dce8',
          cornerRadius: [3, 3, 0, 0]
        }" />

        <!-- Yuqori yog'du chizig'i (highlight) -->
        <v-rect :config="{
          x: 30, y: -12, width: 26, height: 4,
          fill: '#e8f0f8',
          cornerRadius: [3, 3, 0, 0],
          listening: false
        }" />

        <!-- Quyosh qalqoni/visor (eng yuqorida) -->
        <v-rect :config="{
          x: 25, y: -18, width: 30, height: 7,
          fill: '#a0b8cc',
          cornerRadius: [3, 3, 0, 0]
        }" />

        <!-- Old qopqoq (+X tomon, coverage tomoni) -->
        <v-rect :config="{
          x: 55, y: -13, width: 4, height: 26,
          fill: '#253040',
          cornerRadius: [0, 2, 2, 0]
        }" />

        <!-- Status LED -->
        <v-circle :config="{
          x: 32, y: -8,
          radius: 2,
          fill: isSelected ? '#00e676' : '#2ecc71',
          listening: false
        }" />

      </v-group>
    </v-group>

    <!-- ═══ TOOLTIP (Removed, now global) ═══ -->
  </v-group>
</template>
