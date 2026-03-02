<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch, nextTick, computed } from 'vue'
import ParkingArea from './components/ParkingArea.vue'
import SlotEdit from './components/SlotEdit.vue'

// Transformer ref olib tashlandi (mainRect uchun endi kerak emas)
const configKonva = ref({
  width: 0,
  height: 0,
  scaleX: 1,
  scaleY: 1,
  x: 0,
  y: 0,
  textScale: 1
})

const configDeleteButton = ref({
  x: 0,
  y: 0,
  radius: 10,
  fill: 'red',
  visible: false,
  name: 'deleteButton',
  scaleX: 1,
  scaleY: 1
})

const configDeleteText = ref({
  text: '×',
  fontSize: 16,
  fill: 'white',
  width: 20,
  height: 20,
  align: 'center',
  verticalAlign: 'middle',
  visible: false,
  listening: false,
  scaleX: 1,
  scaleY: 1
})

watch(() => configKonva.value.textScale, (newScale) => {
  configDeleteButton.value.scaleX = newScale
  configDeleteButton.value.scaleY = newScale
  
  configDeleteText.value.scaleX = newScale
  configDeleteText.value.scaleY = newScale
})

const containerRef = ref<HTMLElement | null>(null)

const updateSize = () => {
  if (containerRef.value) {
    configKonva.value.width = window.innerWidth
    configKonva.value.height = window.innerHeight
  }
}

onMounted(() => {
  updateSize()
  loadFromLocalStorage()
  window.addEventListener('resize', updateSize)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateSize)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
})

const handleKeyDown = (e: KeyboardEvent) => {
  if (e.ctrlKey && e.key === 's') {
    e.preventDefault()
    requestSave()
    return
  }
  if (e.code === 'Space') {
    isSpacePressed.value = true
    if (containerRef.value) {
      containerRef.value.style.cursor = 'grab'
    }
    // Prevent scrolling with space
    if (e.target === document.body) {
      e.preventDefault()
    }
  }
  if (e.ctrlKey) {
    if (containerRef.value && !isSpacePressed.value) {
      containerRef.value.style.cursor = 'grab'
    }
  }
}

const handleKeyUp = (e: KeyboardEvent) => {
  if (e.code === 'Space') {
    isSpacePressed.value = false
    if (containerRef.value) {
      containerRef.value.style.cursor = e.ctrlKey ? 'grab' : 'default'
    }
  }
  if (e.key === 'Control') {
    if (containerRef.value && !isSpacePressed.value) {
      containerRef.value.style.cursor = 'default'
    }
  }
}

const configRect = ref({
  x: 0,
  y: 0,
  width: 0,
  height: 0,
  fill: 'red',
  opacity: 0.5,
  visible: false,
  listening: false
})

const isDrawing = ref(false)
const isDrawn = ref(false)
const isHovered = ref(false)
const selectedShapeName = ref('')
const PIXELS_PER_METER = 50

const isSpacePressed = ref(false)
const isMiddleMouseDown = ref(false)
const isPanning = ref(false)

// LocalStorage kaliti
const STORAGE_KEY = 'sp-edge-state-v1'

// --- Saqlash mexanizmi ---
const saveConfirmVisible = ref(false)
const saveToastVisible = ref(false)
let saveToastTimer: ReturnType<typeof setTimeout> | null = null

const requestSave = () => {
  saveConfirmVisible.value = true
}

const confirmSave = () => {
  saveConfirmVisible.value = false
  saveToLocalStorage()
  saveToastVisible.value = true
  if (saveToastTimer) clearTimeout(saveToastTimer)
  saveToastTimer = setTimeout(() => { saveToastVisible.value = false }, 2000)
}

const cancelSave = () => {
  saveConfirmVisible.value = false
}

const saveToLocalStorage = () => {
  try {
    const data = {
      isDrawn: isDrawn.value,
      configRect: {
        x: configRect.value.x,
        y: configRect.value.y,
        width: configRect.value.width,
        height: configRect.value.height
      },
      parkingParams: { ...parkingParams.value },
      slotParams: { ...slotParams.value },
      parkingSlotAreas: parkingSlotAreas.value.map(a => ({
        ...a,
        // Funksiyasiz sodda obyekt bo'lishi uchun aynan kerakli maydonlarni saqlaymiz
        config: {
          x: a.config.x,
          y: a.config.y,
          width: a.config.width,
          height: a.config.height,
          rotation: a.config.rotation || 0
        },
        slotCount: a.slotCount,
        slotWidth: a.slotWidth,
        slotHeight: a.slotHeight,
        slotAngle: a.slotAngle,
        pixelsPerMeter: a.pixelsPerMeter || PIXELS_PER_METER,
        id: a.id
      }))
    }
    localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
  } catch (e) {
    console.warn('Saqlashda xatolik:', e)
  }
}

const loadFromLocalStorage = () => {
  try {
    const raw = localStorage.getItem(STORAGE_KEY)
    if (!raw) return
    const data = JSON.parse(raw)

    if (data.parkingParams) {
      parkingParams.value = { ...parkingParams.value, ...data.parkingParams }
    }
    if (data.slotParams) {
      slotParams.value = { ...slotParams.value, ...data.slotParams }
    }

    if (data.configRect) {
      configRect.value.x = Number(data.configRect.x) || 0
      configRect.value.y = Number(data.configRect.y) || 0
      configRect.value.width = Number(data.configRect.width) || 0
      configRect.value.height = Number(data.configRect.height) || 0
      configRect.value.visible = !!data.isDrawn
    }

    if (typeof data.isDrawn === 'boolean') {
      isDrawn.value = data.isDrawn
    }

    if (data.parkingSlotAreas) {
      parkingSlotAreas.value = data.parkingSlotAreas.map((a: any) => {
        const ppm = a.pixelsPerMeter || PIXELS_PER_METER
        const sw = (a.slotWidth || 0) * ppm
        const computedWidth = (a.slotCount || 0) * sw
        const rot = Number(a.config?.rotation ?? 0)
        return {
          id: a.id,
          config: {
            x: Number(a.config?.x) || 0,
            y: Number(a.config?.y) || 0,
            width: Number(a.config?.width) || computedWidth,
            height: Number(a.config?.height) || (a.slotHeight || 0) * ppm,
            rotation: rot,
            draggable: true,
            name: 'slotArea'
          },
          slotCount: Number(a.slotCount) || 0,
          slotWidth: Number(a.slotWidth) || 0,
          slotHeight: Number(a.slotHeight) || 0,
          slotAngle: Number(a.slotAngle ?? 0) || 0,
          pixelsPerMeter: ppm
        }
      })
    }

    // Delete tugmasi pozitsiyalarini qayta hisoblash
    if (isDrawn.value) {
      const w = configRect.value.width
      configDeleteButton.value.x = configRect.value.x + w - 20
      configDeleteButton.value.y = configRect.value.y + 20
      configDeleteText.value.x = configRect.value.x + w - 30
      configDeleteText.value.y = configRect.value.y + 10
    }

    // UI holatini tozalash
    selectedShapeName.value = ''
    editingSlotAreaId.value = null
  } catch (e) {
    console.warn('Yuklashda xatolik:', e)
  }
}

// Slot Area parametrlari
const slotParams = ref({
  count: 5,
  width: 2.5,
  height: 5,
  angle: 0
})

const parkingSlotAreas = ref<any[]>([])
const selectedSlotAreaId = ref<string | null>(null)
const editingSlotAreaId = ref<string | null>(null)
const editPopupPos = ref({ x: 0, y: 0 })

const editingSlotArea = computed(() => {
  if (!editingSlotAreaId.value) return null
  return parkingSlotAreas.value.find(a => a.id === editingSlotAreaId.value) || null
})

const mode = ref<'PARKING' | 'SLOT_AREA'>('PARKING')

const addSlotArea = () => {
  if (!isDrawn.value) return
  
  const id = `slot-area-${Date.now()}`
  const sw = slotParams.value.width * PIXELS_PER_METER
  const sh = slotParams.value.height * PIXELS_PER_METER
  const totalWidth = slotParams.value.count * sw

  const newArea = {
    id,
    config: {
      x: 50, // Default boshlang'ich nuqta (parking ichida)
      y: 50,
      width: totalWidth,
      height: sh,
      rotation: slotParams.value.angle || 0,
      draggable: true,
      name: 'slotArea'
    },
    slotCount: slotParams.value.count,
    slotWidth: slotParams.value.width,
    slotHeight: slotParams.value.height,
    slotAngle: slotParams.value.angle,
    pixelsPerMeter: PIXELS_PER_METER
  }
  
  parkingSlotAreas.value.push(newArea)
  selectedSlotAreaId.value = id
  
  // Avtomatik ravishda tahrirlash oynasini ochish olib tashlandi
  // editingSlotAreaId.value = id
  
  // Oynaning o'rtasiga yaqinroq joyda ochish
  editPopupPos.value = { x: window.innerWidth / 2 - 100, y: 100 }
}

const handleDeleteSlotArea = (id: string) => {
  parkingSlotAreas.value = parkingSlotAreas.value.filter(a => a.id !== id)
  if (selectedSlotAreaId.value === id) {
    selectedSlotAreaId.value = null
  }
}

const handleRotateSlotArea = (id: string) => {
  const area = parkingSlotAreas.value.find(a => a.id === id)
  if (area) {
    area.config.rotation = (area.config.rotation + 90) % 360
    // area.slotAngle endi alohida, uni rotation bilan sinxronlamaymiz
  }
}

const handleUpdateSlotAreaParams = (data: any) => {
  const area = parkingSlotAreas.value.find(a => a.id === data.id)
  if (area) {
    if (data.x !== undefined) area.config.x = data.x
    if (data.y !== undefined) area.config.y = data.y
    if (data.slotCount !== undefined) area.slotCount = data.slotCount
    if (data.slotWidth !== undefined) area.slotWidth = data.slotWidth
    if (data.slotHeight !== undefined) area.slotHeight = data.slotHeight
    if (data.slotAngle !== undefined) {
      area.slotAngle = data.slotAngle
      // area.config.rotation = data.slotAngle  <-- buni o'chirib tashlaymiz
    }
    
    // Hisoblangan qiymatlarni yangilash (NaN dan himoyalangan)
    const sw = (area.slotWidth || 0) * (area.pixelsPerMeter || 50)
    area.config.width = (area.slotCount || 0) * sw
    area.config.height = (area.slotHeight || 0) * (area.pixelsPerMeter || 50)
  }
}

const handleEditSlotArea = (data: { id: string, x: number, y: number, screenX?: number, screenY?: number }) => {
  editingSlotAreaId.value = data.id
  if (data.screenX !== undefined && data.screenY !== undefined) {
    // Popup oynaning o'lchamlarini hisobga olib, ekran chetidan chiqib ketmasligini ta'minlash
    const popupWidth = 250
    const popupHeight = 300
    
    let x = data.screenX + 20
    let y = data.screenY - 50 // Tugmaning biroz yuqorisidan boshlash
    
    if (x + popupWidth > window.innerWidth) {
      x = data.screenX - popupWidth - 20
    }
    
    if (y + popupHeight > window.innerHeight) {
      y = window.innerHeight - popupHeight - 20
    }
    
    if (y < 0) y = 20
    
    editPopupPos.value = { x, y }
  } else {
    editPopupPos.value = { x: 250, y: 20 }
  }
}

// Parking elementlari uchun test ma'lumotlari
const parkingSlots = ref([])

const parkingSlotGroups = ref([])

const resetDrawing = () => {
  isDrawn.value = false
  configRect.value.visible = false
  configDeleteButton.value.visible = false
  configDeleteText.value.visible = false
  selectedShapeName.value = ''
  mode.value = 'PARKING'
  
  // Reset values
  configRect.value.width = 0
  configRect.value.height = 0
  parkingSlotAreas.value = []
  editingSlotAreaId.value = null
}

const updateDimensions = (rectNode: any) => {
  const absW = rectNode.width() * rectNode.scaleX()
  const absH = rectNode.height() * rectNode.scaleY()
  const minX = rectNode.x()
  const minY = rectNode.y()

  // Update original config positions to keep children in sync
  configRect.value.x = minX
  configRect.value.y = minY
  configRect.value.width = absW
  configRect.value.height = absH

  configDeleteButton.value.x = minX + absW - 20
  configDeleteButton.value.y = minY + 20
  configDeleteText.value.x = minX + absW - 30
  configDeleteText.value.y = minY + 10
}

const parkingParams = ref({
  width: 10,
  height: 5
})

const updateParkingDimensions = () => {
  if (!isDrawn.value) return
  
  const w = parkingParams.value.width * PIXELS_PER_METER
  const h = parkingParams.value.height * PIXELS_PER_METER
  
  configRect.value.width = w
  configRect.value.height = h
}

const createParking = () => {
  const w = parkingParams.value.width * PIXELS_PER_METER
  const h = parkingParams.value.height * PIXELS_PER_METER
  
  configRect.value.x = 50
  configRect.value.y = 50
  configRect.value.width = w
  configRect.value.height = h
  configRect.value.visible = true
  
  isDrawn.value = true
  selectedShapeName.value = 'mainRect'

  configDeleteButton.value.x = 50 + w - 20
  configDeleteButton.value.y = 70
  configDeleteText.value.x = 50 + w - 30
  configDeleteText.value.y = 60
}

const handleWheel = (e: any) => {
  e.evt.preventDefault()

  if (e.evt.shiftKey) {
    configKonva.value.x -= e.evt.deltaY
    return
  }

  const scaleBy = 1.1
  const stage = e.target.getStage()
  const oldScale = stage.scaleX()
  const pointer = stage.getPointerPosition()

  if (!pointer) return

  const mousePointTo = {
    x: (pointer.x - stage.x()) / oldScale,
    y: (pointer.y - stage.y()) / oldScale
  }

  const newScale = e.evt.deltaY > 0 ? oldScale / scaleBy : oldScale * scaleBy

  configKonva.value.scaleX = newScale
  configKonva.value.scaleY = newScale
  configKonva.value.textScale = 1 / newScale

  const newPos = {
    x: pointer.x - mousePointTo.x * newScale,
    y: pointer.y - mousePointTo.y * newScale
  }
  configKonva.value.x = newPos.x
  configKonva.value.y = newPos.y
}

const handleStageDragMove = (e: any) => {
  // Stage drag bo'lganda e.target stage ning o'zi bo'ladi
  const stage = e.target.getStage()
  if (e.target === stage) {
    configKonva.value.x = e.target.x()
    configKonva.value.y = e.target.y()
    
    if (isSpacePressed.value || isMiddleMouseDown.value || isPanning.value) {
      if (containerRef.value) {
        containerRef.value.style.cursor = 'grabbing'
      }
    }
  }
}

const handleStageDragEnd = (e: any) => {
  const stage = e.target.getStage()
  if (e.target === stage) {
    configKonva.value.x = e.target.x()
    configKonva.value.y = e.target.y()
    
    if (isSpacePressed.value) {
      if (containerRef.value) {
        containerRef.value.style.cursor = 'grab'
      }
    } else {
      if (containerRef.value) {
        containerRef.value.style.cursor = 'default'
      }
    }
  }
}

const handleMouseDown = (e: any) => {
  const stage = e.target.getStage()
  const targetName = e.target.name()
  const targetType = e.target.getType()

  // Middle mouse button check (button 1)
  if (e.evt && e.evt.button === 1) {
    isMiddleMouseDown.value = true
    stage.startDrag()
    if (containerRef.value) {
      containerRef.value.style.cursor = 'grabbing'
    }
    return
  }

  // Space yoki Ctrl held panning
  if (isSpacePressed.value || (e.evt && e.evt.ctrlKey)) {
    isPanning.value = true
    stage.startDrag()
    if (containerRef.value) {
      containerRef.value.style.cursor = 'grabbing'
    }
    return
  }

  if (isDrawn.value) {
    const isStageClicked = e.target === stage || targetName === 'background'
    
    if (isStageClicked) {
      selectedShapeName.value = ''
      editingSlotAreaId.value = null
    } else {
      let node = e.target
      let name = node.name()
      
      // SlotArea ichidagi elementlar bosilganda ham slotArea tanlanishi kerak
      while (node && name !== 'slotArea' && node.getParent()) {
        node = node.getParent()
        name = node.name()
      }

      if (name === 'slotArea') {
        // Hozircha slotArea uchun transformer yo'q, lekin uni tanlangan qilib belgilashimiz mumkin (kelajak uchun)
        selectedShapeName.value = '' 
        selectedSlotAreaId.value = node.id()
      } else {
        selectedShapeName.value = ''
        // Agar boshqa narsa (masalan mainRect) bosilsa, edit panelni yopish
        editingSlotAreaId.value = null
      }
    }
  }
}

const handleMouseMove = () => {
  // Mouse bilan chizish olib tashlandi
}

const handleMouseUp = (e: any) => {
  const stage = e.target.getStage()
  
  if (isMiddleMouseDown.value || isPanning.value) {
    isMiddleMouseDown.value = false
    isPanning.value = false
    stage.stopDrag()
    if (containerRef.value) {
      const isStillPanningKey = isSpacePressed.value || (e.evt && e.evt.ctrlKey)
      containerRef.value.style.cursor = isStillPanningKey ? 'grab' : 'default'
    }
  }
}

const handleTransform = (e: any) => {
  updateDimensions(e.target)
}

const handleDragMove = (e: any) => {
  updateDimensions(e.target)
}

const handleMouseEnter = () => {
  // Parking mouse eventlari olib tashlandi
}

const handleMouseLeave = () => {
  // Parking mouse eventlari olib tashlandi
}

const zoomPercent = computed(() => Math.round(configKonva.value.scaleX * 100))

const zoomBy = (factor: number) => {
  const oldScale = configKonva.value.scaleX
  const newScale = oldScale * factor
  const cx = configKonva.value.width / 2
  const cy = configKonva.value.height / 2
  const pointTo = {
    x: (cx - configKonva.value.x) / oldScale,
    y: (cy - configKonva.value.y) / oldScale
  }
  configKonva.value.scaleX = newScale
  configKonva.value.scaleY = newScale
  configKonva.value.textScale = 1 / newScale
  configKonva.value.x = cx - pointTo.x * newScale
  configKonva.value.y = cy - pointTo.y * newScale
}

const resetZoom = () => {
  configKonva.value.scaleX = 1
  configKonva.value.scaleY = 1
  configKonva.value.textScale = 1
  configKonva.value.x = 0
  configKonva.value.y = 0
}
</script>

<template>
  <div class="controls">
    <div class="mode-selector">
      <button :class="{ active: mode === 'PARKING' }" @click="mode = 'PARKING'">Parking sozlash</button>
      <button :class="{ active: mode === 'SLOT_AREA' }" @click="mode = 'SLOT_AREA'" :disabled="!isDrawn">Slot Area sozlash</button>
    </div>
    <div v-if="mode === 'PARKING'" class="params">
      <label>Width (m): <input type="number" v-model.number="parkingParams.width" step="0.1" @input="updateParkingDimensions" /></label>
      <label>Height (m): <input type="number" v-model.number="parkingParams.height" step="0.1" @input="updateParkingDimensions" /></label>
      <button v-if="!isDrawn" @click="createParking" class="draw-btn">Chizish</button>
    </div>
    <div v-if="mode === 'SLOT_AREA'" class="params">
      <label>Slot soni: <input type="number" v-model.number="slotParams.count" /></label>
      <label>Width (m): <input type="number" v-model.number="slotParams.width" step="0.1" /></label>
      <label>Height (m): <input type="number" v-model.number="slotParams.height" step="0.1" /></label>
      <label>Angle (°): <input type="number" v-model.number="slotParams.angle" /></label>
      <button @click="addSlotArea" class="draw-btn">Qo'shish</button>
    </div>
  </div>

  <!-- O'ng yuqori burchak: saqlash tugmasi -->
  <div class="top-right-controls">
    <button class="save-btn" @click="requestSave" title="Saqlash (Ctrl+S)">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/>
        <polyline points="17 21 17 13 7 13 7 21"/>
        <polyline points="7 3 7 8 15 8"/>
      </svg>
      Saqlash
    </button>
  </div>

  <!-- Saqlash tasdiqlash dialogi -->
  <div v-if="saveConfirmVisible" class="modal-backdrop" @click.self="cancelSave">
    <div class="modal">
      <p>O'zgarishlarni saqlaysizmi?</p>
      <div class="modal-actions">
        <button class="modal-btn modal-btn--cancel" @click="cancelSave">Bekor qilish</button>
        <button class="modal-btn modal-btn--confirm" @click="confirmSave">Saqlash</button>
      </div>
    </div>
  </div>

  <!-- Saqlandi xabari -->
  <div v-if="saveToastVisible" class="save-toast">Saqlandi ✓</div>
  <div class="zoom-controls">
    <button class="zoom-btn" @click="zoomBy(1 / 1.2)" title="Kichraytirish">−</button>
    <button class="zoom-level" @click="resetZoom" title="100% ga qaytarish">{{ zoomPercent }}%</button>
    <button class="zoom-btn" @click="zoomBy(1.2)" title="Kattalashtirish">+</button>
  </div>
  <div class="container" ref="containerRef">
    <v-stage 
      :config="configKonva"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @wheel="handleWheel"
      @dragmove="handleStageDragMove"
      @dragend="handleStageDragEnd"
      draggable
    >
      <v-layer>
        <!-- Fon to'rtburchagi - stage surish uchun -->
        <v-rect :config="{
          width: 100000,
          height: 100000,
          x: -50000,
          y: -50000,
          fill: '#f0f0f0',
          listening: true,
          name: 'background'
        }" />
        <ParkingArea
          :config="configRect"
          :slots="parkingSlots"
          :slotGroups="parkingSlotGroups"
          :slotAreas="parkingSlotAreas"
          :isDrawn="isDrawn"
          :pixelsPerMeter="PIXELS_PER_METER"
          :textScale="configKonva.textScale"
          @mouseenter="handleMouseEnter"
          @mouseleave="handleMouseLeave"
          @delete-slot-area="handleDeleteSlotArea"
          @rotate-slot-area="handleRotateSlotArea"
          @edit-slot-area="handleEditSlotArea"
          @update-slot-area-position="handleUpdateSlotAreaParams"
        />
      </v-layer>
    </v-stage>
    <SlotEdit
      v-if="editingSlotArea"
      :id="editingSlotArea.id"
      :x="editingSlotArea.config.x"
      :y="editingSlotArea.config.y"
      :slotCount="editingSlotArea.slotCount"
      :slotWidth="editingSlotArea.slotWidth"
      :slotHeight="editingSlotArea.slotHeight"
      :slotAngle="editingSlotArea.slotAngle"
      :pixelsPerMeter="PIXELS_PER_METER"
      :visible="!!editingSlotAreaId"
      :screenX="editPopupPos.x"
      :screenY="editPopupPos.y"
      @update-params="handleUpdateSlotAreaParams"
      @close="editingSlotAreaId = null"
      @mousedown.stop
    />
  </div>
</template>

<style>
body {
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #f0f0f0;
}

.container {
  width: 100vw;
  height: 100vh;
  background-color: white;
}

.controls {
  position: fixed;
  top: 20px;
  left: 20px;
  z-index: 100;
  background: white;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.2);
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.mode-selector {
  display: flex;
  gap: 5px;
}

.mode-selector button {
  padding: 8px 12px;
  cursor: pointer;
  border: 1px solid #ccc;
  background: #f9f9f9;
}

.mode-selector button.active {
  background: #4caf50;
  color: white;
  border-color: #4caf50;
}

.params {
  display: flex;
  flex-direction: column;
  gap: 5px;
  font-size: 14px;
}

.params label {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
}

.params input {
  width: 60px;
}

.draw-btn {
  margin-top: 10px;
  padding: 8px;
  background: #2196f3;
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 4px;
}

.draw-btn:hover {
  background: #1976d2;
}

.zoom-controls {
  position: fixed;
  bottom: 24px;
  right: 24px;
  z-index: 100;
  display: flex;
  align-items: center;
  gap: 2px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.2);
  padding: 4px;
}

.zoom-btn {
  width: 32px;
  height: 32px;
  border: none;
  background: transparent;
  font-size: 20px;
  line-height: 1;
  cursor: pointer;
  border-radius: 6px;
  color: #333;
  display: flex;
  align-items: center;
  justify-content: center;
}

.zoom-btn:hover {
  background: #f0f0f0;
}

.zoom-level {
  min-width: 52px;
  height: 32px;
  border: none;
  background: transparent;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  border-radius: 6px;
  color: #333;
}

.zoom-level:hover {
  background: #f0f0f0;
}

.top-right-controls {
  position: fixed;
  top: 20px;
  right: 24px;
  z-index: 100;
}

.save-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  background: #2196f3;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(33,150,243,0.3);
}

.save-btn:hover {
  background: #1976d2;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  z-index: 200;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal {
  background: white;
  border-radius: 8px;
  padding: 24px 28px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.2);
  min-width: 240px;
  text-align: center;
}

.modal p {
  margin: 0 0 20px;
  font-size: 15px;
  color: #333;
}

.modal-actions {
  display: flex;
  gap: 10px;
  justify-content: center;
}

.modal-btn {
  padding: 8px 20px;
  border: none;
  border-radius: 5px;
  font-size: 14px;
  cursor: pointer;
}

.modal-btn--cancel {
  background: #f0f0f0;
  color: #555;
}

.modal-btn--cancel:hover {
  background: #e0e0e0;
}

.modal-btn--confirm {
  background: #2196f3;
  color: white;
}

.modal-btn--confirm:hover {
  background: #1976d2;
}

.save-toast {
  position: fixed;
  bottom: 72px;
  right: 24px;
  z-index: 300;
  background: #323232;
  color: white;
  font-size: 13px;
  padding: 8px 16px;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.25);
  pointer-events: none;
}
</style>
