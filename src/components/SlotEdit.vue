<script setup lang="ts">
import { computed, ref, watch } from 'vue'

const props = defineProps<{
  id: string
  x: number
  y: number
  slotCount: number
  slotWidth: number
  slotHeight: number
  slotAngle: number
  pixelsPerMeter: number
  visible: boolean
  screenX: number
  screenY: number
}>()

const emit = defineEmits(['update-params', 'close'])

const xMeters = ref(0)
const yMeters = ref(0)
const count = ref(0)
const width = ref(0)
const height = ref(0)
const angle = ref(0)

// Props o'zgarganda ichki qiymatlarni yangilash
watch(() => [props.id, props.x, props.y, props.slotCount, props.slotWidth, props.slotHeight, props.slotAngle], () => {
  xMeters.value = Number((props.x / props.pixelsPerMeter).toFixed(2))
  yMeters.value = Number((props.y / props.pixelsPerMeter).toFixed(2))
  count.value = props.slotCount
  width.value = props.slotWidth
  height.value = props.slotHeight
  angle.value = props.slotAngle
}, { immediate: true })

const handleUpdate = () => {
  emit('update-params', {
    id: props.id,
    x: xMeters.value * props.pixelsPerMeter,
    y: yMeters.value * props.pixelsPerMeter,
    slotCount: count.value,
    slotWidth: width.value,
    slotHeight: height.value,
    slotAngle: angle.value
  })
}

const handleClose = () => {
  emit('close')
}
</script>

<template>
  <div v-if="visible" class="slot-edit-overlay" :style="{ left: screenX + 'px', top: screenY + 'px' }">
    <div class="slot-edit-header">
      <span>Edit Slot Area</span>
      <button @click="handleClose" class="close-btn">×</button>
    </div>
    <div class="slot-edit-body">
      <div class="input-group">
        <label for="se-x">X pozitsiya (m)</label>
        <input id="se-x" type="number" v-model.number="xMeters" step="0.01" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label for="se-y">Y pozitsiya (m)</label>
        <input id="se-y" type="number" v-model.number="yMeters" step="0.01" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label for="se-count">Slotlar soni</label>
        <input id="se-count" type="number" v-model.number="count" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label for="se-width">Slot eni (m)</label>
        <input id="se-width" type="number" v-model.number="width" step="0.1" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label for="se-height">Slot bo'yi (m)</label>
        <input id="se-height" type="number" v-model.number="height" step="0.1" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label for="se-angle">Qiyalik burchagi (°)</label>
        <input id="se-angle" type="number" v-model.number="angle" @input="handleUpdate" />
      </div>
    </div>
  </div>
</template>

<style scoped>
.slot-edit-overlay {
  position: fixed;
  background: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.2);
  padding: 8px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 220px;
  pointer-events: auto;
}

.slot-edit-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  font-weight: bold;
  border-bottom: 1px solid #eee;
  padding-bottom: 4px;
}

.close-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 16px;
  line-height: 1;
  padding: 0;
  color: #666;
}

.slot-edit-body {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.input-group {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.input-group label {
  font-size: 12px;
  flex-shrink: 0;
  color: #555;
  cursor: pointer;
}

.input-group input {
  width: 70px;
  font-size: 12px;
  padding: 2px 4px;
}
</style>
