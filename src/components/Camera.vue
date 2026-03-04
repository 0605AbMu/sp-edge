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
}>()

const emit = defineEmits(['select', 'update-body', 'update-corner', 'delete'])

const isBodyHovered = ref(false)
const hoveredCorner = ref<number | null>(null)

// Drag boshlanganda body va corners ning snapshot'i
const dragSnapshot = ref<{ bodyX: number; bodyY: number; corners: number[] } | null>(null)

const handleDragStart = () => {
  dragSnapshot.value = {
    bodyX: props.x,
    bodyY: props.y,
    corners: [...props.corners]
  }
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
}

const centroid = computed(() => ({
  x: (props.corners[0] + props.corners[2] + props.corners[4] + props.corners[6]) / 4,
  y: (props.corners[1] + props.corners[3] + props.corners[5] + props.corners[7]) / 4
}))

const facingAngle = computed(() =>
  Math.atan2(centroid.value.y - props.y, centroid.value.x - props.x)
)

// Icon ichidagi yo'nalish vektori (local koords, icon rotate bo'lmaydi)
const arrowDx = computed(() => Math.cos(facingAngle.value))
const arrowDy = computed(() => Math.sin(facingAngle.value))

const bodyTooltipText = computed(() => {
  const p = props.pixelsPerMeter
  return `Kamera: (${(props.x / p).toFixed(1)}m, ${(props.y / p).toFixed(1)}m)`
})

const cornerTooltip = (i: number) => {
  const p = props.pixelsPerMeter
  return `B${i + 1}: (${(props.corners[i * 2] / p).toFixed(1)}m, ${(props.corners[i * 2 + 1] / p).toFixed(1)}m)`
}

// Kamera ikonasi uchun ranglar
const bodyColor = computed(() =>
  isBodyHovered.value ? '#1565c0' : (props.isSelected ? '#0d47a1' : '#1976d2')
)

// Icon group uchun dragBoundFunc — parent = Camera root group (parking origin)
const iconDragBound = function (this: any, pos: { x: number; y: number }) {
  const parentPos = this.getParent().getAbsolutePosition()
  const scale = this.getStage().scaleX()
  let lx = (pos.x - parentPos.x) / scale
  let ly = (pos.y - parentPos.y) / scale
  lx = Math.max(0, Math.min(props.parkingWidth, lx))
  ly = Math.max(0, Math.min(props.parkingHeight, ly))
  return { x: lx * scale + parentPos.x, y: ly * scale + parentPos.y }
}

// Burchak tutqichlari uchun dragBoundFunc
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

    <!-- ═══ COVERAGE: faqat aktiv holda ko'rinadi ═══ -->
    <template v-if="isSelected">

      <v-line :config="{
        points: corners,
        fill: 'rgba(33,150,243,0.12)',
        stroke: 'rgba(33,150,243,0.5)',
        strokeWidth: 1,
        closed: true,
        listening: false
      }" />

      <v-line v-for="i in 4" :key="'ray-' + i" :config="{
        points: [x, y, corners[(i - 1) * 2], corners[(i - 1) * 2 + 1]],
        stroke: 'rgba(33,150,243,0.3)',
        strokeWidth: 0.8,
        dash: [5, 4],
        listening: false
      }" />

      <!-- Burchak tutqichlari -->
      <template v-for="i in 4" :key="'c-' + i">
        <v-circle :config="{
          x: corners[(i - 1) * 2],
          y: corners[(i - 1) * 2 + 1],
          radius: 5,
          fill: hoveredCorner === i - 1 ? '#f57c00' : 'white',
          stroke: '#f57c00',
          strokeWidth: 2,
          draggable: true,
          scaleX: textScale,
          scaleY: textScale,
          dragBoundFunc: cornerDragBound,
          name: 'cameraCorner'
        }"
          @mouseenter="hoveredCorner = i - 1"
          @mouseleave="hoveredCorner = null"
          @dragend="(e: any) => handleCornerDragEnd(e, i - 1)"
        />

        <v-label v-if="hoveredCorner === i - 1" :config="{
          x: corners[(i - 1) * 2],
          y: corners[(i - 1) * 2 + 1] - 16 * textScale,
          scaleX: textScale,
          scaleY: textScale
        }">
          <v-tag :config="{ fill: '#e65100', opacity: 0.9, pointerDirection: 'down', pointerWidth: 5, pointerHeight: 5, lineJoin: 'round', cornerRadius: 3 }" />
          <v-text :config="{ text: cornerTooltip(i - 1), fontSize: 11, fill: 'white', padding: 5, listening: false }" />
        </v-label>
      </template>

    </template>

    <!-- ═══ KAMERA IKONASI — masshtab ta'sir qilmaydi ═══ -->
    <v-group
      :config="{
        x, y,
        scaleX: textScale,
        scaleY: textScale,
        draggable: true,
        dragBoundFunc: iconDragBound,
        name: 'cameraBody'
      }"
      @mouseenter="isBodyHovered = true"
      @mouseleave="isBodyHovered = false"
      @click="emit('select', id)"
      @dragstart="handleDragStart"
      @dragend="handleDragEnd"
    >
      <!-- Tanlov halqasi (active) -->
      <v-rect v-if="isSelected" :config="{
        x: -25, y: -24, width: 50, height: 38,
        fill: 'transparent',
        stroke: '#90caf9',
        strokeWidth: 1.5,
        cornerRadius: 8,
        dash: [5, 3],
        listening: false
      }" />

      <!-- Yo'nalish o'qi -->
      <v-line :config="{
        points: [arrowDx * 22, arrowDy * 22, arrowDx * 40, arrowDy * 40],
        stroke: isSelected ? '#90caf9' : 'rgba(255,255,255,0.8)',
        strokeWidth: 2,
        listening: false
      }" />

      <!-- Kamera tanasi (asosiy korpus) -->
      <v-rect :config="{
        x: -22, y: -13, width: 44, height: 26,
        fill: bodyColor,
        cornerRadius: 5,
        shadowColor: 'rgba(0,0,0,0.35)',
        shadowBlur: 4,
        shadowOffset: { x: 0, y: 2 }
      }" />

      <!-- Vizörnik (tepadagi kichik bo'rtma) -->
      <v-rect :config="{
        x: -7, y: -19, width: 14, height: 7,
        fill: bodyColor,
        cornerRadius: 2
      }" />

      <!-- Ob'ektiv — tashqi halqa -->
      <v-circle :config="{
        x: 0, y: 0,
        radius: 11,
        fill: '#1a237e',
        stroke: 'rgba(255,255,255,0.25)',
        strokeWidth: 1.5
      }" />

      <!-- Ob'ektiv — o'rta qatlam -->
      <v-circle :config="{
        x: 0, y: 0,
        radius: 7.5,
        fill: '#0d0d2b'
      }" />

      <!-- Ob'ektiv — markaziy nuqta -->
      <v-circle :config="{
        x: 0, y: 0,
        radius: 3.5,
        fill: '#263238'
      }" />

      <!-- Ob'ektiv — yorug'lik aks -->
      <v-circle :config="{
        x: -3, y: -3,
        radius: 2.5,
        fill: 'rgba(255,255,255,0.32)',
        listening: false
      }" />

      <!-- Chap tomon tugmachasi (detal) -->
      <v-rect :config="{
        x: -22, y: -6, width: 4, height: 8,
        fill: 'rgba(255,255,255,0.15)',
        cornerRadius: 1,
        listening: false
      }" />
    </v-group>

    <!-- ═══ TOOLTIP — hover bo'lganda ═══ -->
    <v-label v-if="isBodyHovered" :config="{
      x,
      y: y - 28 * textScale,
      scaleX: textScale,
      scaleY: textScale
    }">
      <v-tag :config="{
        fill: '#0d47a1', opacity: 0.9,
        pointerDirection: 'down', pointerWidth: 6, pointerHeight: 6,
        lineJoin: 'round', cornerRadius: 4
      }" />
      <v-text :config="{
        text: bodyTooltipText,
        fontSize: 11, fill: 'white', padding: 6, listening: false
      }" />
    </v-label>

    <!-- ═══ O'CHIRISH TUGMASI ═══ -->
    <v-group v-if="isBodyHovered" :config="{
      x: x + 26 * textScale,
      y: y - 22 * textScale,
      scaleX: textScale,
      scaleY: textScale
    }">
      <v-circle :config="{ x: 0, y: 0, radius: 12, fill: 'white', stroke: 'red' }" @click="emit('delete', id)" />
      <v-text :config="{ text: '×', x: -7, y: -9, fontSize: 18, fill: 'red', listening: false }" />
    </v-group>

  </v-group>
</template>
