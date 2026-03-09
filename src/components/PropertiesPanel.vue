<script setup lang="ts">
import { computed, ref, watch } from 'vue'

interface CameraData {
  id: string
  x: number
  y: number
  corners: number[]
}

interface SlotAreaData {
  id: string
  config: { x: number; y: number; rotation?: number }
  slotCount: number
  slotWidth: number
  slotHeight: number
  slotAngle: number
}

interface GateData {
  id: string
  side: 'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT'
  x: number
  y: number
  width: number
  type: 'ENTRANCE' | 'EXIT' | 'BOTH'
  isOpen?: boolean
  rotation?: number
}

interface RoadData {
  id: string
  x: number
  y: number
  width: number
  height: number
  rotation: number
  name: string
}

interface ParkingAreaData {
  width: number
  height: number
}

const props = defineProps<{
  camera: CameraData | null
  slotArea: SlotAreaData | null
  gate: GateData | null
  road: RoadData | null
  parkingArea: ParkingAreaData | null
  pixelsPerMeter: number
  isReadonly?: boolean
}>()

const emit = defineEmits([
  'update-camera',
  'update-slot-area',
  'update-gate',
  'update-road',
  'update-parking-area',
  'delete-camera',
  'delete-slot-area',
  'delete-gate',
  'delete-road',
  'rotate-slot-area',
  'clone-camera',
  'clone-slot-area',
  'clone-gate',
  'clone-road',
])

// ─── ParkingArea local state ──────────────────────────────────────────────────
const paWidth = ref(50)
const paHeight = ref(30)

watch(() => props.parkingArea, (pa) => {
  if (!pa) return
  paWidth.value = pa.width
  paHeight.value = pa.height
}, { immediate: true })

const applyParkingUpdate = () => {
  emit('update-parking-area', {
    width: paWidth.value,
    height: paHeight.value
  })
}

// ─── Camera local state ───────────────────────────────────────────────────────
const camX = ref(0)
const camY = ref(0)
const camCorners = ref<number[]>([])

watch(() => props.camera, (c) => {
  if (!c) return
  const p = props.pixelsPerMeter
  camX.value = parseFloat((c.x / p).toFixed(2))
  camY.value = parseFloat((c.y / p).toFixed(2))
  camCorners.value = [...c.corners.map((v) => parseFloat((v / p).toFixed(2)))]
}, { immediate: true })

const applyCameraUpdate = () => {
  if (!props.camera) return
  const p = props.pixelsPerMeter
  const corners = camCorners.value.map(v => v * p)
  emit('update-camera', {
    id: props.camera.id,
    x: camX.value * p,
    y: camY.value * p,
    corners
  })
}

// ─── SlotArea local state ─────────────────────────────────────────────────────
const saX = ref(0)
const saY = ref(0)
const saRotation = ref(0)
const saCount = ref(1)
const saWidth = ref(2.5)
const saHeight = ref(5)
const saAngle = ref(0)

watch(() => props.slotArea, (a) => {
  if (!a) return
  const p = props.pixelsPerMeter
  saX.value = parseFloat((a.config.x / p).toFixed(2))
  saY.value = parseFloat((a.config.y / p).toFixed(2))
  saRotation.value = a.config.rotation || 0
  saCount.value = a.slotCount
  saWidth.value = a.slotWidth
  saHeight.value = a.slotHeight
  saAngle.value = a.slotAngle
}, { immediate: true })

const applySlotAreaUpdate = () => {
  if (!props.slotArea) return
  const p = props.pixelsPerMeter
  emit('update-slot-area', {
    id: props.slotArea.id,
    x: saX.value * p,
    y: saY.value * p,
    rotation: saRotation.value,
    slotCount: saCount.value,
    slotWidth: saWidth.value,
    slotHeight: saHeight.value,
    slotAngle: saAngle.value
  })
}

// ─── Gate local state ─────────────────────────────────────────────────────────
const gateType = ref<'ENTRANCE' | 'EXIT' | 'BOTH'>('ENTRANCE')
const gateSide = ref<'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT'>('TOP')
const gateX = ref(0)
const gateY = ref(0)
const gateWidth = ref(3)
const gateIsOpen = ref(false)
const gateRotation = ref(0)

watch(() => props.gate, (g) => {
  if (!g) return
  const p = props.pixelsPerMeter
  gateType.value = g.type
  gateSide.value = g.side
  gateX.value = parseFloat((g.x / p).toFixed(2))
  gateY.value = parseFloat((g.y / p).toFixed(2))
  gateWidth.value = g.width
  gateIsOpen.value = !!g.isOpen
  gateRotation.value = g.rotation || 0
}, { immediate: true })

const applyGateUpdate = () => {
  if (!props.gate) return
  const p = props.pixelsPerMeter
  emit('update-gate', {
    id: props.gate.id,
    type: gateType.value,
    side: gateSide.value,
    x: gateX.value * p,
    y: gateY.value * p,
    width: gateWidth.value,
    isOpen: gateIsOpen.value,
    rotation: gateRotation.value
  })
}

const toggleGate = () => {
  gateIsOpen.value = !gateIsOpen.value
  applyGateUpdate()
}

// ─── Road local state ─────────────────────────────────────────────────────────
const roadX = ref(0)
const roadY = ref(0)
const roadWidth = ref(6)
const roadHeight = ref(10)
const roadRotation = ref(0)
const roadName = ref('')

watch(() => props.road, (r) => {
  if (!r) return
  const p = props.pixelsPerMeter
  roadX.value = parseFloat((r.x / p).toFixed(2))
  roadY.value = parseFloat((r.y / p).toFixed(2))
  roadWidth.value = r.width
  roadHeight.value = r.height
  roadRotation.value = r.rotation
  roadName.value = r.name
}, { immediate: true })

const applyRoadUpdate = () => {
  if (!props.road) return
  const p = props.pixelsPerMeter
  emit('update-road', {
    id: props.road.id,
    x: roadX.value * p,
    y: roadY.value * p,
    width: roadWidth.value,
    height: roadHeight.value,
    rotation: roadRotation.value,
    name: roadName.value
  })
}

// ─── Active panel ─────────────────────────────────────────────────────────────
const activeType = computed(() => {
  if (props.camera) return 'camera'
  if (props.slotArea) return 'slotArea'
  if (props.gate) return 'gate'
  if (props.road) return 'road'
  if (props.parkingArea) return 'parkingArea'
  return null
})

const cornerLabel = (i: number) => ['B1', 'B2', 'B3', 'B4'][i]
</script>

<template>
  <div v-if="activeType" class="props-panel" @mousedown.stop>
    <div class="panel-header">
      <span class="panel-title">
        <template v-if="activeType === 'camera'">Kamera</template>
        <template v-else-if="activeType === 'slotArea'">Slot maydon</template>
        <template v-else-if="activeType === 'road'">Yo'lak</template>
        <template v-else-if="activeType === 'parkingArea'">Parking maydon</template>
        <template v-else>Eshik</template>
      </span>
      <button
        v-if="!isReadonly && activeType !== 'parkingArea'"
        class="delete-btn"
        @click="activeType === 'camera' ? emit('delete-camera', camera!.id)
               : activeType === 'slotArea' ? emit('delete-slot-area', slotArea!.id)
               : activeType === 'road' ? emit('delete-road', road!.id)
               : emit('delete-gate', gate!.id)"
        title="O'chirish"
      >×</button>
    </div>

    <!-- ── PARKING AREA ─────────────────────────────────────────────────── -->
    <template v-if="activeType === 'parkingArea' && parkingArea">
      <div class="section-label">O'lchamlar</div>
      <div class="prop-row">
        <span class="prop-name">Eni (m)</span>
        <input class="prop-input" type="number" step="0.5" v-model.number="paWidth" @change="applyParkingUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Bo'yi (m)</span>
        <input class="prop-input" type="number" step="0.5" v-model.number="paHeight" @change="applyParkingUpdate" :disabled="isReadonly" />
      </div>
      
      <p class="hint-text-mini" style="margin-top: 12px; color: #888;">
        Parking maydoni o'lchamlarini o'zgartirish orqali asosiy chizilgan maydonni kengaytirishingiz yoki qisqartirishingiz mumkin.
      </p>
    </template>

    <!-- ── CAMERA ────────────────────────────────────────────────────────── -->
    <template v-if="activeType === 'camera' && camera">
      <div class="section-label">Pozitsiya</div>
      <div class="prop-row">
        <span class="prop-name">X (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="camX" @change="applyCameraUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Y (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="camY" @change="applyCameraUpdate" :disabled="isReadonly" />
      </div>

      <div class="section-label">Kuzatuv burchaklari</div>
      <template v-for="i in 4" :key="'corner-' + i">
        <div class="prop-row corner-header">
          <span class="corner-label">{{ cornerLabel(i - 1) }}</span>
        </div>
        <div class="prop-row">
          <span class="prop-name indent">X (m)</span>
          <input class="prop-input" type="number" step="0.1"
            :value="camCorners[(i - 1) * 2]"
            :disabled="isReadonly"
            @change="(e: any) => { camCorners[(i-1)*2] = parseFloat(e.target.value); applyCameraUpdate() }" />
        </div>
        <div class="prop-row">
          <span class="prop-name indent">Y (m)</span>
          <input class="prop-input" type="number" step="0.1"
            :value="camCorners[(i - 1) * 2 + 1]"
            :disabled="isReadonly"
            @change="(e: any) => { camCorners[(i-1)*2+1] = parseFloat(e.target.value); applyCameraUpdate() }" />
        </div>
      </template>

      <div v-if="!isReadonly" class="section-label">Amallar</div>
      <div v-if="!isReadonly" class="prop-row actions-row">
        <button class="action-btn clone-btn" @click="emit('clone-camera', camera.id)" :disabled="isReadonly" title="Klonlash">
          <span class="icon">👯</span>
          <span>Klonlash</span>
        </button>
        <button class="action-btn delete-btn-large" @click="emit('delete-camera', camera.id)" title="O'chirish">
          <span class="icon">×</span>
          <span>O'chirish</span>
        </button>
      </div>
    </template>

    <!-- ── SLOT AREA ─────────────────────────────────────────────────────── -->
    <template v-else-if="activeType === 'slotArea' && slotArea">
      <div class="section-label">Pozitsiya</div>
      <div class="prop-row">
        <span class="prop-name">X (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="saX" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Y (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="saY" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Burish (°)</span>
        <input class="prop-input" type="number" step="90" v-model.number="saRotation" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>

      <div class="section-label">Slot sozlamalari</div>
      <div class="prop-row">
        <span class="prop-name">Soni</span>
        <input class="prop-input" type="number" min="1" step="1" v-model.number="saCount" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Kenglik (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="saWidth" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Uzunlik (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="saHeight" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Burchak (°)</span>
        <input class="prop-input" type="number" step="1" v-model.number="saAngle" @change="applySlotAreaUpdate" :disabled="isReadonly" />
      </div>

      <div v-if="!isReadonly" class="section-label">Amallar</div>
      <div v-if="!isReadonly" class="prop-row actions-row">
        <button class="action-btn clone-btn" @click="emit('clone-slot-area', slotArea.id)" :disabled="isReadonly" title="Klonlash">
          <span class="icon">👯</span>
          <span>Klonlash</span>
        </button>
        <button class="action-btn rotate-btn" @click="emit('rotate-slot-area', slotArea.id)" title="90° burish">
          <span class="icon">↻</span>
          <span>Burish</span>
        </button>
        <button class="action-btn delete-btn-large" @click="emit('delete-slot-area', slotArea.id)" title="O'chirish">
          <span class="icon">×</span>
          <span>O'chirish</span>
        </button>
      </div>
    </template>

    <!-- ── GATE ──────────────────────────────────────────────────────────── -->
    <template v-else-if="activeType === 'gate' && gate">
      <div class="section-label">Eshik sozlamalari</div>
      <div class="prop-row">
        <span class="prop-name">Turi</span>
        <select class="prop-select" v-model="gateType" @change="applyGateUpdate" :disabled="isReadonly">
          <option value="ENTRANCE">Kirish</option>
          <option value="EXIT">Chiqish</option>
          <option value="BOTH">Kirish/Chiqish</option>
        </select>
      </div>
      <div class="prop-row">
        <span class="prop-name">Tomon</span>
        <select class="prop-select" v-model="gateSide" @change="applyGateUpdate" :disabled="isReadonly">
          <option value="TOP">Yuqori</option>
          <option value="BOTTOM">Pastki</option>
          <option value="LEFT">Chap</option>
          <option value="RIGHT">O'ng</option>
        </select>
      </div>
      <div class="prop-row">
        <span class="prop-name">Kenglik (m)</span>
        <input class="prop-input" type="number" step="0.5" min="1" v-model.number="gateWidth" @change="applyGateUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Burish (°)</span>
        <div class="prop-input-group">
          <input class="prop-input" type="number" step="5" v-model.number="gateRotation" @change="applyGateUpdate" :disabled="isReadonly" />
          <input type="range" min="0" max="360" step="1" v-model.number="gateRotation" @input="applyGateUpdate" class="prop-slider" :disabled="isReadonly" />
        </div>
      </div>

      <div class="section-label">Pozitsiya</div>
      <div class="prop-row">
        <span class="prop-name">X (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="gateX" @change="applyGateUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Y (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="gateY" @change="applyGateUpdate" :disabled="isReadonly" />
      </div>

      <div v-if="!isReadonly" class="section-label">Amallar</div>
      <div v-if="!isReadonly" class="prop-row actions-row">
        <button class="action-btn clone-btn" @click="emit('clone-gate', gate.id)" :disabled="isReadonly" title="Klonlash">
          <span class="icon">👯</span>
          <span>Klonlash</span>
        </button>
        <button class="action-btn toggle-gate-btn" :class="{ 'is-open': gateIsOpen }" @click="toggleGate" :disabled="isReadonly">
          <span class="icon">{{ gateIsOpen ? '🔓' : '🔒' }}</span>
          <span>{{ gateIsOpen ? 'Yopish' : 'Ochish' }}</span>
        </button>
        <button class="action-btn delete-btn-large" @click="emit('delete-gate', gate.id)" title="O'chirish" :disabled="isReadonly">
          <span class="icon">×</span>
          <span>O'chirish</span>
        </button>
      </div>
    </template>

    <!-- ── ROAD ──────────────────────────────────────────────────────────── -->
    <template v-else-if="activeType === 'road' && road">
      <div class="section-label">Umumiy</div>
      <div class="prop-row">
        <span class="prop-name">Nomi</span>
        <input class="prop-input" type="text" v-model="roadName" @change="applyRoadUpdate" :disabled="isReadonly" />
      </div>
      
      <div class="section-label">Pozitsiya</div>
      <div class="prop-row">
        <span class="prop-name">X (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="roadX" @change="applyRoadUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Y (m)</span>
        <input class="prop-input" type="number" step="0.1" v-model.number="roadY" @change="applyRoadUpdate" :disabled="isReadonly" />
      </div>
      <div class="prop-row">
        <span class="prop-name">Burish (°)</span>
        <div class="prop-input-group">
          <input class="prop-input" type="number" step="5" v-model.number="roadRotation" @change="applyRoadUpdate" :disabled="isReadonly" />
          <input type="range" min="0" max="360" step="1" v-model.number="roadRotation" @input="applyRoadUpdate" class="prop-slider" :disabled="isReadonly" />
        </div>
      </div>

      <div class="section-label">O'lchamlar</div>
      <div class="prop-row">
        <span class="prop-name">Kenglik (m)</span>
        <div class="prop-input-group">
          <input class="prop-input" type="number" min="1" max="100" step="0.1" v-model.number="roadWidth" @change="applyRoadUpdate" :disabled="isReadonly" />
          <input type="range" min="1" max="50" step="0.5" v-model.number="roadWidth" @input="applyRoadUpdate" class="prop-slider" :disabled="isReadonly" />
        </div>
      </div>
      <div class="prop-row">
        <span class="prop-name">Balandlik (m)</span>
        <input class="prop-input" type="number" min="1" max="20" step="0.1" v-model.number="roadHeight" @change="applyRoadUpdate" :disabled="isReadonly" />
      </div>

      <div v-if="!isReadonly" class="section-label">Amallar</div>
      <div v-if="!isReadonly" class="prop-row actions-row">
        <button class="action-btn clone-btn" @click="emit('clone-road', road.id)" :disabled="isReadonly" title="Klonlash">
          <span class="icon">👯</span>
          <span>Klonlash</span>
        </button>
        <button class="action-btn delete-btn-large" @click="emit('delete-road', road.id)" title="O'chirish" :disabled="isReadonly">
          <span class="icon">×</span>
          <span>O'chirish</span>
        </button>
      </div>
    </template>

    <!-- ── BROWSER PREVIEW ────────────────────────────────────────────────── -->
    <div class="browser-preview-section">
      <div class="section-label">O'tish</div>
      <div class="browser-window">
        <div class="browser-header">
          <div class="browser-dots">
            <span class="dot red"></span>
            <span class="dot yellow"></span>
            <span class="dot green"></span>
          </div>
          <div class="browser-address-bar">
            <span class="lock-icon">🔒</span>
            <span class="address-text">https://camera-stream.local/view/{{ activeType }}-{{ camera?.id || slotArea?.id || gate?.id || road?.id }}</span>
          </div>
        </div>
        <div class="browser-content">
          <div class="stream-placeholder">
            <div class="live-badge">LIVE</div>
            <div class="stream-icon">
              <template v-if="activeType === 'camera'">📹</template>
              <template v-else-if="activeType === 'slotArea'">🚗</template>
              <template v-else-if="activeType === 'gate'">🚪</template>
              <template v-else>🛣️</template>
            </div>
            <div class="stream-text">Video oqim yuklanmoqda...</div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.props-panel {
  width: 100%;
  background: transparent;
  font-size: 12px;
  color: #c0c0c0;
  user-select: none;
  border: none;
  box-shadow: none;
  border-radius: 0;
}

.props-panel::-webkit-scrollbar {
  width: 6px;
}
.props-panel::-webkit-scrollbar-track { background: #111; }
.props-panel::-webkit-scrollbar-thumb { background: #444; border-radius: 3px; }

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px 10px;
  border-bottom: 1px solid #2e2e2e;
  background: #1e1e1e;
  border-radius: 0;
}

.panel-title {
  font-size: 13px;
  font-weight: 700;
  color: #f5c518;
  letter-spacing: 0.3px;
}

.delete-btn {
  width: 22px;
  height: 22px;
  border: 1px solid #ff4444;
  background: transparent;
  color: #ff4444;
  border-radius: 4px;
  cursor: pointer;
  font-size: 15px;
  line-height: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
}
.delete-btn:hover {
  background: #ff4444;
  color: #fff;
}

.section-label {
  background: #222;
  color: #f5c518;
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 0.8px;
  text-transform: uppercase;
  padding: 5px 12px 4px;
  border-top: 1px solid #2e2e2e;
  border-bottom: 1px solid #2a2a2a;
}

.prop-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 3px 12px;
  border-bottom: 1px solid #222;
  min-height: 26px;
}

.prop-row:hover {
  background: #1f1f1f;
}

.prop-name {
  color: #888;
  flex: 1;
  white-space: nowrap;
}

.prop-name.indent {
  padding-left: 10px;
  color: #666;
}

.corner-header {
  background: #1e1e1e;
  padding: 4px 12px;
  border-bottom: 1px solid #2a2a2a;
}

.corner-label {
  font-size: 11px;
  font-weight: 700;
  color: #ff9800;
}

.prop-input {
  width: 80px;
  background: #2a2a2a;
  border: 1px solid #3a3a3a;
  color: #e8e8e8;
  border-radius: 3px;
  padding: 2px 6px;
  font-size: 12px;
  text-align: right;
}

.prop-input-group {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  justify-content: flex-end;
}

.prop-slider {
  flex: 1;
  max-width: 80px;
  height: 4px;
  accent-color: #f1c40f;
  cursor: pointer;
}

.prop-input:focus {
  outline: none;
  border-color: #f5c518;
  background: #2f2f2f;
}

.prop-select {
  width: 86px;
  background: #2a2a2a;
  border: 1px solid #3a3a3a;
  color: #e8e8e8;
  border-radius: 3px;
  padding: 2px 4px;
  font-size: 12px;
}

.prop-select:focus {
  outline: none;
  border-color: #f5c518;
}

.prop-value-static {
  color: #777;
  font-size: 12px;
  text-align: right;
}

.actions-row {
  display: flex;
  gap: 8px;
  padding: 8px 12px;
  border-bottom: none;
}

.action-btn {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  padding: 6px 8px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
}

.action-btn .icon {
  font-size: 14px;
  line-height: 1;
}

.rotate-btn {
  background: transparent;
  border: 1px solid #00bfff;
  color: #00bfff;
}

.rotate-btn:hover {
  background: #00bfff;
  color: #fff;
}

.clone-btn {
  background: transparent;
  border: 1px solid #9c27b0;
  color: #9c27b0;
}

.clone-btn:hover {
  background: #9c27b0;
  color: #fff;
}

.toggle-gate-btn {
  background: transparent;
  border: 1px solid #ff9800;
  color: #ff9800;
}

.toggle-gate-btn:hover {
  background: rgba(255, 152, 0, 0.1);
}

.toggle-gate-btn.is-open {
  border-color: #4caf50;
  color: #4caf50;
}

.toggle-gate-btn.is-open:hover {
  background: rgba(76, 175, 80, 0.1);
}

.delete-btn-large {
  background: transparent;
  border: 1px solid #ff4444;
  color: #ff4444;
}

.delete-btn-large:hover {
  background: #ff4444;
  color: #fff;
}

/* Browser Preview Styles */
.browser-preview-section {
  padding: 10px 0;
  border-top: 1px solid #2e2e2e;
}

.browser-window {
  margin: 10px 12px;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 6px;
  overflow: hidden;
  box-shadow: 0 4px 15px rgba(0,0,0,0.4);
}

.browser-header {
  height: 28px;
  background: #2a2a2a;
  display: flex;
  align-items: center;
  padding: 0 10px;
  gap: 12px;
  border-bottom: 1px solid #333;
}

.browser-dots {
  display: flex;
  gap: 6px;
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.dot.red { background: #ff5f56; }
.dot.yellow { background: #ffbd2e; }
.dot.green { background: #27c93f; }

.browser-address-bar {
  flex: 1;
  height: 18px;
  background: #111;
  border-radius: 3px;
  display: flex;
  align-items: center;
  padding: 0 8px;
  gap: 6px;
  font-size: 9px;
  color: #666;
  overflow: hidden;
  white-space: nowrap;
}

.lock-icon {
  font-size: 8px;
}

.address-text {
  opacity: 0.8;
}

.browser-content {
  aspect-ratio: 16 / 9;
  background: #000;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
}

.stream-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.live-badge {
  position: absolute;
  top: 8px;
  right: 8px;
  background: #ff4444;
  color: #fff;
  font-size: 9px;
  font-weight: 700;
  padding: 2px 6px;
  border-radius: 3px;
  animation: pulse 2s infinite;
}

.stream-icon {
  font-size: 24px;
  opacity: 0.5;
}

.stream-text {
  font-size: 10px;
  color: #555;
  font-style: italic;
}

@keyframes pulse {
  0% { opacity: 1; }
  50% { opacity: 0.6; }
  100% { opacity: 1; }
}
</style>
