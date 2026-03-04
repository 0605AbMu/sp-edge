<script setup lang="ts">
import { ref, watch } from 'vue'

const props = defineProps<{
  id: string
  x: number
  y: number
  rotation: number
  fov: number
  range: number
  pixelsPerMeter: number
  visible: boolean
  screenX: number
  screenY: number
}>()

const emit = defineEmits(['update-params', 'close'])

const xM = ref(0)
const yM = ref(0)
const rot = ref(0)
const fovAngle = ref(90)
const rangeM = ref(10)

watch(
  () => [props.id, props.x, props.y, props.rotation, props.fov, props.range],
  () => {
    xM.value = Number((props.x / props.pixelsPerMeter).toFixed(2))
    yM.value = Number((props.y / props.pixelsPerMeter).toFixed(2))
    rot.value = props.rotation
    fovAngle.value = props.fov
    rangeM.value = props.range
  },
  { immediate: true }
)

const handleUpdate = () => {
  emit('update-params', {
    id: props.id,
    x: xM.value * props.pixelsPerMeter,
    y: yM.value * props.pixelsPerMeter,
    rotation: rot.value,
    fov: fovAngle.value,
    range: rangeM.value
  })
}
</script>

<template>
  <div v-if="visible" class="camera-edit" :style="{ left: screenX + 'px', top: screenY + 'px' }">
    <div class="camera-edit-header">
      <span>Kamera sozlamalari</span>
      <button @click="emit('close')" class="close-btn">×</button>
    </div>
    <div class="camera-edit-body">
      <div class="input-group">
        <label>X (m)</label>
        <input type="number" v-model.number="xM" step="0.1" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label>Y (m)</label>
        <input type="number" v-model.number="yM" step="0.1" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label>Yo'nalish (°)</label>
        <input type="number" v-model.number="rot" step="1" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label>Ko'rish burchagi (°)</label>
        <input type="number" v-model.number="fovAngle" step="5" min="10" max="360" @input="handleUpdate" />
      </div>
      <div class="input-group">
        <label>Masofa (m)</label>
        <input type="number" v-model.number="rangeM" step="0.5" min="1" @input="handleUpdate" />
      </div>
    </div>
  </div>
</template>

<style scoped>
.camera-edit {
  position: fixed;
  background: white;
  border: 1px solid #90caf9;
  border-radius: 6px;
  box-shadow: 0 2px 12px rgba(25, 118, 210, 0.2);
  padding: 8px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 210px;
  pointer-events: auto;
}

.camera-edit-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  font-weight: bold;
  color: #1565c0;
  border-bottom: 1px solid #e3f2fd;
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

.camera-edit-body {
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
}

.input-group input {
  width: 70px;
  font-size: 12px;
  padding: 2px 4px;
}
</style>
