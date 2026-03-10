<script setup lang="ts">
import { ref, shallowRef, onMounted, onUnmounted, watch, nextTick, computed } from 'vue'
import ParkingArea from './components/ParkingArea.vue'
import RoadArea from './components/RoadArea.vue'
import Camera from './components/Camera.vue'
import Gate from './components/Gate.vue'
import PropertiesPanel from './components/PropertiesPanel.vue'
import Ruler from './components/Ruler.vue'

// Transformer ref olib tashlandi (mainRect uchun endi kerak emas)
const configKonva = ref({
  width: window.innerWidth - 280,
  height: window.innerHeight,
  scaleX: 1,
  scaleY: 1,
  x: 280, // Sidebar width
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
  x: 0,
  y: 0,
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

const currentMousePos = ref({ x: 0, y: 0 })
const showMouseIndicators = ref(true)

watch(() => configKonva.value.textScale, (newScale) => {
  configDeleteButton.value.scaleX = newScale
  configDeleteButton.value.scaleY = newScale
  
  configDeleteText.value.scaleX = newScale
  configDeleteText.value.scaleY = newScale
})

const containerRef = ref<HTMLElement | null>(null)

// --- State and Interfaces ---
interface CameraData {
  id: string
  x: number       // parking-relative pixels
  y: number
  corners: number[] // [x1,y1, x2,y2, x3,y3, x4,y4]
}

interface GateData {
  id: string
  side: 'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT'
  x: number
  y: number
  width: number  // meters
  type: 'ENTRANCE' | 'EXIT' | 'BOTH'
  isOpen?: boolean
  rotation?: number
}

interface RoadData {
  id: string
  x: number
  y: number
  width: number  // meters
  height: number // meters
  rotation: number
  name: string
}

const cameras = shallowRef<CameraData[]>([])
const selectedCameraId = ref<string | null>(null)
const editingCameraId = ref<string | null>(null)

const parkingSlotAreas = shallowRef<any[]>([])
const selectedSlotAreaId = ref<string | null>(null)

const gates = shallowRef<GateData[]>([])
const selectedGateId = ref<string | null>(null)

const roads = shallowRef<RoadData[]>([])
const selectedRoadId = ref<string | null>(null)

const isParkingSelected = ref(false)

// Properties panel: selected component data
const selectedCamera = computed(() =>
  selectedCameraId.value ? cameras.value.find(c => c.id === selectedCameraId.value) || null : null
)
const selectedSlotArea = computed(() =>
  selectedSlotAreaId.value ? parkingSlotAreas.value.find(a => a.id === selectedSlotAreaId.value) || null : null
)
const selectedGate = computed(() =>
  selectedGateId.value ? gates.value.find(g => g.id === selectedGateId.value) || null : null
)
const selectedRoad = computed(() =>
  roads.value.find(r => r.id === selectedRoadId.value) || null
)
const selectedParkingArea = computed(() =>
  isParkingSelected.value && isDrawn.value ? parkingParams.value : null
)

const isRightSidebarVisible = computed(() => {
  const visible = !!(selectedCamera.value || selectedSlotArea.value || selectedGate.value || selectedRoad.value || selectedParkingArea.value)
  return visible
})

const updateSize = () => {
  if (containerRef.value) {
    const sidebarWidth = 280
    const rightSidebarWidth = isRightSidebarVisible.value ? 280 : 0
    configKonva.value.width = window.innerWidth - sidebarWidth - rightSidebarWidth
    configKonva.value.height = window.innerHeight
    configKonva.value.x = 0
  }
}

watch(() => isRightSidebarVisible.value, () => {
  nextTick(() => {
    updateSize()
  })
})

onMounted(() => {
  loadFromLocalStorage()
  nextTick(() => {
    updateSize()
  })
  window.addEventListener('resize', updateSize)
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  window.addEventListener('mousedown', handleOutsideClick)
})

onUnmounted(() => {
  window.removeEventListener('resize', updateSize)
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  window.removeEventListener('mousedown', handleOutsideClick)
})

const handleOutsideClick = (e: MouseEvent) => {
  if (!showBottomSheet.value) return
  
  const target = e.target as HTMLElement
  // Agar click bottom-controls ichidan bo'lsa (tugma yoki menu), yopmaymiz
  const controls = document.querySelector('.bottom-controls')
  if (controls && controls.contains(target)) {
    return
  }
  
  showBottomSheet.value = false
}

const handleKeyDown = (e: KeyboardEvent) => {
  if ((e.ctrlKey || e.metaKey) && (e.key === 'c' || e.key === 'C')) {
    // Clone logic for Ctrl+C or Cmd+C
    if (selectedCameraId.value) {
      e.preventDefault()
      handleCloneCamera(selectedCameraId.value)
      return
    }
    if (selectedSlotAreaId.value) {
      e.preventDefault()
      handleCloneSlotArea(selectedSlotAreaId.value)
      return
    }
    if (selectedGateId.value) {
      e.preventDefault()
      handleCloneGate(selectedGateId.value)
      return
    }
    if (selectedRoadId.value) {
      e.preventDefault()
      handleCloneRoad(selectedRoadId.value)
      return
    }
  }

  if ((e.ctrlKey || e.metaKey) && e.key === 's') {
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
  width: 1000,
  height: 600,
  fill: '#2c2c2c',
  opacity: 1,
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
      configKonva: {
        x: configKonva.value.x,
        y: configKonva.value.y,
        scaleX: configKonva.value.scaleX,
        scaleY: configKonva.value.scaleY
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
      })),
      cameras: cameras.value.map(c => ({
        id: c.id,
        x: c.x,
        y: c.y,
        corners: [...c.corners]
      })),
      gates: gates.value.map(g => ({
        id: g.id,
        side: g.side,
        x: g.x,
        y: g.y,
        width: g.width,
        type: g.type,
        rotation: g.rotation || 0
      })),
      roads: (roads.value || []).map(r => ({
        id: r.id,
        x: r.x,
        y: r.y,
        width: r.width,
        height: r.height,
        rotation: r.rotation,
        name: r.name
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
    if (!raw) {
      // Agar xotirada ma'lumot bo'lmasa, barcha state'larni boshlang'ich holatga qaytarish
      parkingSlotAreas.value = []
      cameras.value = []
      gates.value = []
      roads.value = []
      isDrawn.value = false
      configRect.value = { x: 0, y: 0, width: 0, height: 0, fill: '#2c2c2c', opacity: 1, visible: false, listening: false }
      return
    }
    const data = JSON.parse(raw)

    if (data.parkingParams) {
      parkingParams.value = { ...parkingParams.value, ...data.parkingParams }
    }
    if (data.slotParams) {
      slotParams.value = { ...slotParams.value, ...data.slotParams }
    }

    if (data.configRect) {
      configRect.value.x = Number(data.configRect.x) ?? 0
      configRect.value.y = Number(data.configRect.y) ?? 0
      configRect.value.width = Number(data.configRect.width) || 0
      configRect.value.height = Number(data.configRect.height) || 0
      configRect.value.visible = !!data.isDrawn
    }

    if (data.configKonva) {
      configKonva.value.x = Number(data.configKonva.x) ?? 0
      configKonva.value.y = Number(data.configKonva.y) ?? 0
      configKonva.value.scaleX = Number(data.configKonva.scaleX) || 1
      configKonva.value.scaleY = Number(data.configKonva.scaleY) || 1
      configKonva.value.textScale = 1 / (Number(data.configKonva.scaleX) || 1)
    } else {
      configKonva.value.x = 0
      configKonva.value.y = 0
      configKonva.value.scaleX = 1
      configKonva.value.scaleY = 1
      configKonva.value.textScale = 1
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

    if (Array.isArray(data.cameras)) {
      cameras.value = data.cameras.map((c: any) => ({
        id: c.id,
        x: Number(c.x) || 0,
        y: Number(c.y) || 0,
        corners: Array.isArray(c.corners) ? c.corners.map(Number) : []
      }))
    }

    if (Array.isArray(data.gates)) {
      gates.value = data.gates.map((g: any) => ({
        id: g.id,
        side: g.side || 'TOP',
        x: Number(g.x) || 0,
        y: Number(g.y) || 0,
        width: Number(g.width) || 3,
        type: g.type || 'ENTRANCE',
        rotation: Number(g.rotation) || 0
      }))
    }

    if (Array.isArray(data.roads)) {
      roads.value = data.roads.map((r: any) => ({
        id: r.id,
        x: Number(r.x) || 0,
        y: Number(r.y) || 0,
        width: Number(r.width) || 15,
        height: Number(r.height) || 4,
        rotation: Number(r.rotation) || 0,
        name: r.name || 'Yo\'lak'
      }))
    }

    // UI holatini tozalash
    selectedShapeName.value = ''
    selectedSlotAreaId.value = null
    selectedCameraId.value = null
    selectedGateId.value = null
  } catch (e) {
    console.warn('Yuklashda xatolik:', e)
  }
}

// Barcha elementlarni tozalash
const clearAll = () => {
  if (isReadonly.value) return
  if (!confirm("Barcha elementlarni o'chirib tashlamoqchimisiz?")) return
  
  parkingSlotAreas.value = []
  cameras.value = []
  gates.value = []
  roads.value = []
  isDrawn.value = false
  configRect.value = { ...configRect.value, x: 0, y: 0, width: 0, height: 0, visible: false }
  
  selectedShapeName.value = ''
  selectedSlotAreaId.value = null
  selectedCameraId.value = null
  selectedGateId.value = null
  selectedRoadId.value = null
  isParkingSelected.value = false

  // Xotiradan butunlay tozalash (LocalStorage)
  localStorage.removeItem(STORAGE_KEY)
  
  // State'larni darhol tozalash va load funksiyasini chaqirish orqali
  // xotira va UI hamohangligini ta'minlash
  loadFromLocalStorage()
}

// Slot Area parametrlari
const slotParams = ref({
  count: 5,
  width: 2.5,
  height: 5,
  angle: 0
})

const handlePanelUpdateCamera = (data: any) => {
  handleUpdateCameraBody(data)
}

const handlePanelUpdateSlotArea = (data: any) => {
  const index = parkingSlotAreas.value.findIndex(a => a.id === data.id)
  if (index === -1) return
  
  const area = { ...parkingSlotAreas.value[index] }
  if (data.x !== undefined) area.config.x = data.x
  if (data.y !== undefined) area.config.y = data.y
  if (data.rotation !== undefined) area.config.rotation = data.rotation
  if (data.slotCount !== undefined) area.slotCount = data.slotCount
  if (data.slotWidth !== undefined) area.slotWidth = data.slotWidth
  if (data.slotHeight !== undefined) area.slotHeight = data.slotHeight
  if (data.slotAngle !== undefined) area.slotAngle = data.slotAngle
  
  const sw = (area.slotWidth || 0) * (area.pixelsPerMeter || 50)
  area.config.width = (area.slotCount || 0) * sw
  area.config.height = (area.slotHeight || 0) * (area.pixelsPerMeter || 50)
  
  const newAreas = [...parkingSlotAreas.value]
  newAreas[index] = area
  parkingSlotAreas.value = newAreas
}

const handlePanelUpdateGate = (data: any) => {
  const index = gates.value.findIndex(g => g.id === data.id)
  if (index === -1) return
  
  const gate = { ...gates.value[index] }
  if (data.type !== undefined) gate.type = data.type
  if (data.width !== undefined) gate.width = data.width
  if (data.isOpen !== undefined) gate.isOpen = data.isOpen
  if (data.rotation !== undefined) gate.rotation = data.rotation
  if (data.x !== undefined) gate.x = data.x
  if (data.y !== undefined) gate.y = data.y
  
  if (data.side !== undefined && data.side !== gate.side) {
    gate.side = data.side
    switch (gate.side) {
      case 'TOP':    gate.x = configRect.value.width / 2; gate.y = 0; break
      case 'BOTTOM': gate.x = configRect.value.width / 2; gate.y = configRect.value.height; break
      case 'LEFT':   gate.x = 0; gate.y = configRect.value.height / 2; break
      case 'RIGHT':  gate.x = configRect.value.width; gate.y = configRect.value.height / 2; break
    }
  }

  const newGates = [...gates.value]
  newGates[index] = gate
  gates.value = newGates
}

const hoverSlotArea = ref<any>(null)

const handleSlotAreaHoverChange = (data: any) => {
  if (data.isHovered) {
    hoverSlotArea.value = data
  } else if (hoverSlotArea.value?.id === data.id) {
    hoverSlotArea.value = null
  }
}

// --- Yo'lak (Road) state ---

const addRoad = () => {
  if (isReadonly.value) return
  if (!isDrawn.value) return
  const id = `road-${Date.now()}`
  const W = configRect.value.width
  const H = configRect.value.height
  const P = PIXELS_PER_METER
  
  // Create road at the center of the drawn area
  const rWidth = 15
  const rHeight = 4
  
  roads.value = [...roads.value, {
    id,
    x: configRect.value.x + (W / 2) - (rWidth * P / 2),
    y: configRect.value.y + (H / 2) - (rHeight * P / 2),
    width: rWidth,   // default 15m width for horizontal road
    height: rHeight, // default 4m height
    rotation: 0,
    name: 'Yo\'lak'
  }]
  handleSelectRoad(id)
}

const handleDeleteRoad = (id: string) => {
  roads.value = [...roads.value.filter(r => r.id !== id)]
  if (selectedRoadId.value === id) selectedRoadId.value = null
}

const handleSelectRoad = (id: string, shouldCenter: boolean = true) => {
  const isAlreadySelected = selectedRoadId.value === id
  isParkingSelected.value = false
  selectedCameraId.value = null
  selectedSlotAreaId.value = null
  selectedGateId.value = null
  selectedRoadId.value = id
  sidebarTab.value = 'EDIT' // isReadonly dan qat'iy nazar EDIT tabiga o'tamiz
  
  if (!isAlreadySelected && shouldCenter) {
    const road = roads.value.find(r => r.id === id)
    if (road) {
      centerOnElement(road.x, road.y, road.width * PIXELS_PER_METER, road.height * PIXELS_PER_METER)
    }
  }
}

const handleUpdateRoad = (data: Partial<RoadData> & { id: string }) => {
  const roadIndex = roads.value.findIndex(r => r.id === data.id)
  if (roadIndex !== -1) {
    const road = { ...roads.value[roadIndex] }
    if (data.x !== undefined) road.x = data.x
    if (data.y !== undefined) road.y = data.y
    if (data.width !== undefined) road.width = data.width
    if (data.height !== undefined) road.height = data.height
    if (data.rotation !== undefined) road.rotation = data.rotation
    if (data.name !== undefined) road.name = data.name
    
    const newRoads = [...roads.value]
    newRoads[roadIndex] = road
    roads.value = newRoads
  }
}

const handleCloneRoad = (id: string) => {
  if (isReadonly.value) return
  const road = roads.value.find(r => r.id === id)
  if (road) {
    const newRoad = JSON.parse(JSON.stringify(road))
    newRoad.id = `road-${Date.now()}`
    newRoad.x += PIXELS_PER_METER
    newRoad.y += PIXELS_PER_METER
    roads.value.push(newRoad)
    selectedRoadId.value = newRoad.id
  }
}

// --- Kamera state ---

const isReadonly = ref(false)

const toggleReadonly = () => {
  // Avvalgi mantiq tanlovlarni o'chirib yuborar edi, endi buni olib tashlaymiz
  // Chunki element tanlangan holda ham readonly ga o'tish mumkin bo'lishi kerak
}

// --- Eshik (Gate) state ---

const gateParams = ref({
  type: 'ENTRANCE' as 'ENTRANCE' | 'EXIT' | 'BOTH',
  side: 'TOP' as 'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT',
  width: 4
})

const addGate = () => {
  if (isReadonly.value) return
  if (!isDrawn.value) return
  const id = `gate-${Date.now()}`
  const { side, type, width } = gateParams.value
  const fullWidth = width * PIXELS_PER_METER
  let x = 0, y = 0
  switch (side) {
    case 'TOP':    x = configRect.value.width / 2 - fullWidth / 2; y = 0; break
    case 'BOTTOM': x = configRect.value.width / 2 - fullWidth / 2; y = configRect.value.height; break
    case 'LEFT':   x = 0; y = configRect.value.height / 2 - fullWidth / 2; break
    case 'RIGHT':  x = configRect.value.width; y = configRect.value.height / 2 - fullWidth / 2; break
  }
  if (side === 'TOP' || side === 'BOTTOM') {
    x = Math.max(0, Math.min(configRect.value.width - fullWidth, x))
  } else {
    y = Math.max(0, Math.min(configRect.value.height - fullWidth, y))
  }
  gates.value.push({ id, side, x, y, width, type, isOpen: false, rotation: 0 })
  handleSelectGate(id)
}

const handleDeleteGate = (id: string) => {
  gates.value = [...gates.value.filter(g => g.id !== id)]
  if (selectedGateId.value === id) selectedGateId.value = null
}

const handleSelectGate = (id: string, shouldCenter: boolean = true) => {
  const isAlreadySelected = selectedGateId.value === id
  isParkingSelected.value = false
  selectedCameraId.value = null
  selectedSlotAreaId.value = null
  selectedRoadId.value = null
  selectedGateId.value = id
  sidebarTab.value = 'EDIT' // isReadonly dan qat'iy nazar EDIT tabiga o'tamiz
  
  if (!isAlreadySelected && shouldCenter) {
    const gate = gates.value.find(g => g.id === id)
    if (gate) {
      // Eshik o'lchami: kengligi - gate.width, balandligi taxminan 1 metr
      centerOnElement(gate.x, gate.y, gate.width * PIXELS_PER_METER, PIXELS_PER_METER)
    }
  }
}

const handleCloneGate = (id: string) => {
  if (isReadonly.value) return
  const gate = gates.value.find(g => g.id === id)
  if (gate) {
    const newGate = JSON.parse(JSON.stringify(gate))
    newGate.id = `gate-${Date.now()}`
    // Gate tomoniga qarab biroz surish
    if (newGate.side === 'TOP' || newGate.side === 'BOTTOM') {
      newGate.x += PIXELS_PER_METER
    } else {
      newGate.y += PIXELS_PER_METER
    }
    gates.value.push(newGate)
    selectedGateId.value = newGate.id
  }
}

const handleUpdateGatePosition = (data: { id: string; x: number; y: number }) => {
  const gateIndex = gates.value.findIndex(g => g.id === data.id)
  if (gateIndex !== -1) {
    const gate = { ...gates.value[gateIndex] }
    // Gate.vue provides coordinates relative to the same parent as configRect
    // but the Gate component in template uses x + configRect.x
    // so we store coordinates relative to ParkingArea (0..W, 0..H)
    gate.x = data.x - configRect.value.x
    gate.y = data.y - configRect.value.y
    
    const newGates = [...gates.value]
    newGates[gateIndex] = gate
    gates.value = newGates
  }
}

const addCamera = () => {
  if (isReadonly.value) return
  if (!isDrawn.value) return
  const id = `camera-${Date.now()}`
  const W = configRect.value.width
  const H = configRect.value.height
  const cx = W / 2
  const cy = H / 2
  const P = PIXELS_PER_METER
  // Default trapezoid: kamera o'ng tomonga qaragan holda
  const clampX = (v: number) => Math.max(0, Math.min(W, v))
  const clampY = (v: number) => Math.max(0, Math.min(H, v))
  cameras.value.push({
    id, x: cx, y: cy,
    corners: [
      clampX(cx + P * 4.5), clampY(cy - P * 1.5),   // yaqin-yuqori
      clampX(cx + P * 4.5), clampY(cy + P * 1.5),   // yaqin-pastki
      clampX(cx + P * 9),   clampY(cy + P * 3.5),   // uzoq-pastki
      clampX(cx + P * 9),   clampY(cy - P * 3.5),   // uzoq-yuqori
    ]
  })
  handleSelectCamera(id)
}

const handleDeleteCamera = (id: string) => {
  cameras.value = [...cameras.value.filter(c => c.id !== id)]
  if (selectedCameraId.value === id) selectedCameraId.value = null
}

const handleSelectCamera = (id: string, shouldCenter: boolean = true) => {
  const isAlreadySelected = selectedCameraId.value === id
  // Readonly rejimida bo'lganda ham tanlash mumkin
  isParkingSelected.value = false
  selectedSlotAreaId.value = null
  selectedGateId.value = null
  selectedRoadId.value = null
  selectedCameraId.value = id
  sidebarTab.value = 'EDIT' // Har doim EDIT tabiga o'tamiz
  
  if (!isAlreadySelected && shouldCenter) {
    const cam = cameras.value.find(c => c.id === id)
    if (cam) {
      // Kameraning o'lchami taxminan 20px (radius 10)
      centerOnElement(cam.x, cam.y, 20, 20)
    }
  }
}

const handleUpdateCameraBody = (data: { id: string; x: number; y: number; corners?: number[] }) => {
  const camIndex = cameras.value.findIndex(c => c.id === data.id)
  if (camIndex !== -1) {
    const cam = { ...cameras.value[camIndex] }
    const prevX = cam.x
    const prevY = cam.y
    cam.x = data.x - configRect.value.x
    cam.y = data.y - configRect.value.y
    
    if (data.corners) {
      cam.corners = data.corners.map((v, i) => i % 2 === 0 ? v - configRect.value.x : v - configRect.value.y)
    } else {
      // Agar corners berilmagan bo'lsa (faqat tana surilsa), burchaklarni ham siljitish
      const dx = cam.x - prevX
      const dy = cam.y - prevY
      cam.corners = cam.corners.map((v, i) => i % 2 === 0 ? v + dx : v + dy)
    }
    
    const newCameras = [...cameras.value]
    newCameras[camIndex] = cam
    cameras.value = newCameras
  }
}

const handleUpdateCameraCorner = (data: { id: string; index: number; x: number; y: number }) => {
  const camIndex = cameras.value.findIndex(c => c.id === data.id)
  if (camIndex !== -1) {
    const cam = { ...cameras.value[camIndex] }
    cam.corners = [...cam.corners]
    cam.corners[data.index * 2] = data.x - configRect.value.x
    cam.corners[data.index * 2 + 1] = data.y - configRect.value.y
    
    const newCameras = [...cameras.value]
    newCameras[camIndex] = cam
    cameras.value = newCameras
  }
}

const handleCloneCamera = (id: string) => {
  if (isReadonly.value) return
  const cam = cameras.value.find(c => c.id === id)
  if (cam) {
    const newCam = JSON.parse(JSON.stringify(cam))
    newCam.id = `camera-${Date.now()}`
    newCam.x += PIXELS_PER_METER
    newCam.y += PIXELS_PER_METER
    newCam.corners = newCam.corners.map((v: number) => v + PIXELS_PER_METER)
    cameras.value = [...cameras.value, newCam]
    selectedCameraId.value = newCam.id
  }
}

const addSlotArea = () => {
  if (isReadonly.value) return
  if (!isDrawn.value) return
  
  const id = `slot-area-${Date.now()}`
  const sw = slotParams.value.width * PIXELS_PER_METER
  const sh = slotParams.value.height * PIXELS_PER_METER
  const angleRad = (slotParams.value.angle * Math.PI) / 180
  const sOffset = sh * Math.tan(angleRad)
  const totalWidth = slotParams.value.count * sw + Math.abs(sOffset)

  const newArea = {
    id,
    config: {
      x: 50, // Default boshlang'ich nuqta (parking ichida)
      y: 50,
      width: totalWidth,
      height: sh,
      rotation: 0,
      draggable: true,
      name: 'slotArea'
    },
    slotCount: slotParams.value.count,
    slotWidth: slotParams.value.width,
    slotHeight: slotParams.value.height,
    slotAngle: slotParams.value.angle,
    pixelsPerMeter: PIXELS_PER_METER
  }
  
  parkingSlotAreas.value = [...parkingSlotAreas.value, newArea]
  // Yangi SlotArea qo'shilganda avtomatik tanlab, sidebar'da ko'rsatamiz
  handleEditSlotArea(id)
}

const handleDeleteSlotArea = (id: string) => {
  parkingSlotAreas.value = [...parkingSlotAreas.value.filter(a => a.id !== id)]
  if (selectedSlotAreaId.value === id) {
    selectedSlotAreaId.value = null
  }
}

const handleRotateSlotArea = (id: string) => {
  const index = parkingSlotAreas.value.findIndex(a => a.id === id)
  if (index !== -1) {
    const area = { ...parkingSlotAreas.value[index] }
    area.config = { ...area.config }
    area.config.rotation = (area.config.rotation + 90) % 360
    
    const newAreas = [...parkingSlotAreas.value]
    newAreas[index] = area
    parkingSlotAreas.value = newAreas
  }
}

const handleCloneSlotArea = (id: string) => {
  if (isReadonly.value) return
  const area = parkingSlotAreas.value.find(a => a.id === id)
  if (area) {
    const newArea = JSON.parse(JSON.stringify(area))
    newArea.id = `slot-area-${Date.now()}`
    newArea.config.x += PIXELS_PER_METER
    newArea.config.y += PIXELS_PER_METER
    parkingSlotAreas.value = [...parkingSlotAreas.value, newArea]
    selectedSlotAreaId.value = newArea.id
  }
}

const handleUpdateSlotAreaParams = (data: any) => {
  const areaIndex = parkingSlotAreas.value.findIndex(a => a.id === data.id)
  if (areaIndex !== -1) {
    const area = JSON.parse(JSON.stringify(parkingSlotAreas.value[areaIndex]))
    if (data.x !== undefined) area.config.x = data.x
    if (data.y !== undefined) area.config.y = data.y
    if (data.slotCount !== undefined) area.slotCount = data.slotCount
    if (data.slotWidth !== undefined) area.slotWidth = data.slotWidth
    if (data.slotHeight !== undefined) area.slotHeight = data.slotHeight
    if (data.slotAngle !== undefined) {
      area.slotAngle = data.slotAngle
    }
    
    // Hisoblangan qiymatlarni yangilash (NaN dan himoyalangan)
    const sw = (area.slotWidth || 0) * (area.pixelsPerMeter || 50)
    const sh = (area.slotHeight || 0) * (area.pixelsPerMeter || 50)
    const angleRad = ((area.slotAngle || 0) * Math.PI) / 180
    const sOffset = sh * Math.tan(angleRad)
    
    area.config.width = (area.slotCount || 0) * sw + Math.abs(sOffset)
    area.config.height = sh

    const newAreas = [...parkingSlotAreas.value]
    newAreas[areaIndex] = area
    parkingSlotAreas.value = newAreas
  }
}

const handleEditSlotArea = (id: string, shouldCenter: boolean = true) => {
  const isAlreadySelected = selectedSlotAreaId.value === id
  // Slot areani tanlaymiz - sidebar properties panelda ko'rsatiladi
  isParkingSelected.value = false
  selectedCameraId.value = null
  selectedGateId.value = null
  selectedRoadId.value = null
  selectedSlotAreaId.value = id
  sidebarTab.value = 'EDIT' // Har doim EDIT tabiga o'tamiz
  
  if (!isAlreadySelected && shouldCenter) {
    const area = parkingSlotAreas.value.find(a => a.id === id)
    if (area) {
      centerOnElement(area.config.x, area.config.y, area.config.width, area.config.height)
    }
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
  
  // Reset values
  configRect.value.width = 0
  configRect.value.height = 0
  parkingSlotAreas.value = []
  selectedSlotAreaId.value = null
  cameras.value = []
  selectedCameraId.value = null
  gates.value = []
  selectedGateId.value = null
  roads.value = []
  selectedRoadId.value = null
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
  width: 50,
  height: 30
})

const handleSelectParkingArea = (shouldCenter: boolean = true) => {
  const isAlreadySelected = isParkingSelected.value
  selectedCameraId.value = null
  selectedSlotAreaId.value = null
  selectedGateId.value = null
  selectedRoadId.value = null
  isParkingSelected.value = true
  sidebarTab.value = 'EDIT'
  
  if (!isAlreadySelected && shouldCenter) {
    centerOnElement(configRect.value.x, configRect.value.y, configRect.value.width, configRect.value.height)
  }
}

const handlePanelUpdateParkingArea = (data: { width: number; height: number }) => {
  parkingParams.value.width = data.width
  parkingParams.value.height = data.height
  updateParkingDimensions()
}

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
  
  configRect.value.x = 0
  configRect.value.y = 0
  configRect.value.width = w
  configRect.value.height = h
  configRect.value.visible = true
  
  isDrawn.value = true
  selectedShapeName.value = 'mainRect'

  configDeleteButton.value.x = w - 20
  configDeleteButton.value.y = 20
  configDeleteText.value.x = w - 30
  configDeleteText.value.y = 10
}

const handleWheel = (e: any) => {
  e.evt.preventDefault()

  // Ctrl bosilganda — zoom
  if (e.evt.ctrlKey) {
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
    return
  }

  // Odatda scroll — pan (shift: gorizontal, aks holda vertikal)
  if (e.evt.shiftKey) {
    configKonva.value.x -= e.evt.deltaY
  } else {
    // Zoom pivot point should exclude sidebar
    const sidebarW = 280
    if (e.evt.clientX < sidebarW) return // Do not pan if scrolling inside sidebar
    
    configKonva.value.x -= e.evt.deltaX
    configKonva.value.y -= e.evt.deltaY
  }
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
    const isCameraElement = targetName === 'cameraBody' || targetName === 'cameraCorner'

    if (isStageClicked) {
      selectedShapeName.value = ''
      selectedSlotAreaId.value = null
      selectedCameraId.value = null
      selectedGateId.value = null
      selectedRoadId.value = null
      isParkingSelected.value = false
    } else {
      let node = e.target
      let name = node.name()

      // SlotArea ichidagi elementlar bosilganda ham slotArea tanlanishi kerak
      while (node && name !== 'slotArea' && node.getParent()) {
        node = node.getParent()
        name = node.name()
      }

      if (name === 'slotArea') {
        selectedShapeName.value = ''
        selectedSlotAreaId.value = node.id()
        selectedCameraId.value = null
        selectedGateId.value = null
        selectedRoadId.value = null
        isParkingSelected.value = false
      }
    }
  }
}

const handleMouseMove = (e: any) => {
  const stage = e.target.getStage ? e.target.getStage() : e.target
  if (!stage) return
  const mousePos = stage.getPointerPosition()
  if (mousePos) {
    currentMousePos.value = {
      x: mousePos.x,
      y: mousePos.y
    }
  }
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
  handleMouseMove(e)
}

const zoomPercent = computed(() => Math.round(configKonva.value.scaleX * 100))

const zoomBy = (factor: number) => {
  const oldScale = configKonva.value.scaleX
  const newScale = oldScale * factor
  const sidebarW = 280
  const cx = sidebarW + (configKonva.value.width - sidebarW) / 2
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

const centerOnElement = (x: number, y: number, width: number = 0, height: number = 0) => {
  const sidebarW = 280
  const availableWidth = configKonva.value.width - sidebarW
  const availableHeight = configKonva.value.height
  
  // Hozirgi stage o'lchami va pozitsiyasi
  const scale = configKonva.value.scaleX
  const stageX = configKonva.value.x
  const stageY = configKonva.value.y

  // Elementning ekrandagi koordinatalari
  const elementScreenX1 = stageX + x * scale
  const elementScreenY1 = stageY + y * scale
  const elementScreenX2 = elementScreenX1 + width * scale
  const elementScreenY2 = elementScreenY1 + height * scale

  // Viewport chegaralari (ekran koordinatalarida)
  const viewportX1 = sidebarW
  const viewportY1 = 0
  const viewportX2 = sidebarW + availableWidth
  const viewportY2 = availableHeight

  // Element viewport ichida ekanligini tekshirish (to'liq ko'rinayotgan bo'lsa)
  const isVisible = (
    elementScreenX1 >= viewportX1 &&
    elementScreenY1 >= viewportY1 &&
    elementScreenX2 <= viewportX2 &&
    elementScreenY2 <= viewportY2
  )

  // Agar element allaqachon ko'rinayotgan bo'lsa, hech narsani o'zgartirmaymiz
  if (isVisible) {
    return
  }

  // Markaziy nuqtani hisoblash (sahna koordinatalarida)
  const elementCenterX = x + width / 2
  const elementCenterY = y + height / 2
  
  // Viewport markazi (ekran koordinatalarida)
  const viewportCenterX = sidebarW + availableWidth / 2
  const viewportCenterY = availableHeight / 2
  
  // Yangi stage pozitsiyasi
  const newX = viewportCenterX - elementCenterX * scale
  const newY = viewportCenterY - elementCenterY * scale

  // Agar pozitsiya juda yaqin bo'lsa (1px farq), sakrashni oldini olish uchun o'zgartirmaymiz
  if (Math.abs(configKonva.value.x - newX) < 1 && Math.abs(configKonva.value.y - newY) < 1) {
    return
  }

  configKonva.value.x = newX
  configKonva.value.y = newY
}


const sidebarTab = ref('EDIT')

const showBottomSheet = ref(false)
const bottomSheetTab = ref('ADD')

const toggleBottomSheet = (tab: string) => {
  if (showBottomSheet.value && bottomSheetTab.value === tab) {
    showBottomSheet.value = false
  } else {
    bottomSheetTab.value = tab
    showBottomSheet.value = true
  }
}
</script>

<template>
  <!-- Chap sidebar - barcha boshqaruvlar -->
  <div class="left-sidebar">
    <div class="sidebar-header">
      <h3 class="sidebar-title">Elementlar brauzeri</h3>
    </div>
    
    <div class="element-browser">
      <div v-if="!isDrawn && cameras.length === 0 && gates.length === 0 && roads.length === 0" class="empty-state">
        Elementlar yo'q
      </div>
      <div v-else class="element-list">
        <div v-if="isDrawn" class="element-item" :class="{ active: isParkingSelected }" @click="handleSelectParkingArea">
          <span class="element-icon">🅿️</span>
          <span class="element-name">Parking maydon</span>
        </div>
        
        <div v-for="cam in cameras" :key="cam.id" class="element-item" :class="{ active: selectedCameraId === cam.id }" @click="handleSelectCamera(cam.id)">
          <span class="element-icon">📹</span>
          <span class="element-name">Kamera {{ cam.id.slice(-4) }}</span>
        </div>

        <div v-for="sa in parkingSlotAreas" :key="sa.id" class="element-item" :class="{ active: selectedSlotAreaId === sa.id }" @click="handleEditSlotArea(sa.id)">
          <span class="element-icon">🚗</span>
          <span class="element-name">Slot maydon ({{ sa.slotCount }})</span>
        </div>

        <div v-for="gate in gates" :key="gate.id" class="element-item" :class="{ active: selectedGateId === gate.id }" @click="handleSelectGate(gate.id)">
          <span class="element-icon">🚪</span>
          <span class="element-name">Eshik ({{ gate.type }})</span>
        </div>

        <div v-for="road in roads" :key="road.id" class="element-item" :class="{ active: selectedRoadId === road.id }" @click="handleSelectRoad(road.id)">
          <span class="element-icon">🛣️</span>
          <span class="element-name">{{ road.name || 'Yo\'lak' }}</span>
        </div>
      </div>
    </div>
  </div>

  <!-- O'ng sidebar - element tahrirlash -->
  <div v-if="isRightSidebarVisible" class="right-sidebar">
    <div class="sidebar-header">
      <h3 class="sidebar-title">Elementni tahrirlash</h3>
    </div>

    <div class="tab-pane">
      <!-- Properties Panel - har doim render qilinadi -->
      <div class="properties-section">
        <PropertiesPanel
          :camera="selectedCamera"
          :slotArea="selectedSlotArea"
          :gate="selectedGate"
          :road="selectedRoad"
          :parkingArea="selectedParkingArea"
          :pixelsPerMeter="PIXELS_PER_METER"
          :isReadonly="isReadonly"
          @update-camera="handlePanelUpdateCamera"
          @update-slot-area="handlePanelUpdateSlotArea"
          @update-gate="handlePanelUpdateGate"
          @update-road="handleUpdateRoad"
          @update-parking-area="handlePanelUpdateParkingArea"
          @delete-camera="handleDeleteCamera"
          @delete-slot-area="handleDeleteSlotArea"
          @delete-gate="handleDeleteGate"
          @delete-road="handleDeleteRoad"
          @rotate-slot-area="handleRotateSlotArea"
          @clone-camera="handleCloneCamera"
          @clone-slot-area="handleCloneSlotArea"
          @clone-gate="handleCloneGate"
          @clone-road="handleCloneRoad"
        />
      </div>
    </div>
  </div>

  <!-- Pastki boshqaruv tugmalari -->
  <div class="bottom-controls">
    <!-- ADD dropdown -->
    <div class="dropdown-container">
      <button class="bottom-ctrl-btn" :class="{ active: showBottomSheet && bottomSheetTab === 'ADD' }" @click="toggleBottomSheet('ADD')">
        <span>➕ Qo'shish</span>
      </button>
      <div v-if="showBottomSheet && bottomSheetTab === 'ADD'" class="dropdown-menu">
        <div class="dropdown-header">Qo'shish</div>
        <div class="form-row">
          <button v-if="!isDrawn" @click="createParking" :disabled="isReadonly" class="action-btn-primary-mini">🅿️ Parking chizish</button>
          <button @click="addSlotArea" :disabled="!isDrawn || isReadonly" class="action-btn-primary-mini">🚗 Slot qo'shish</button>
          <button @click="addCamera" :disabled="!isDrawn || isReadonly" class="action-btn-primary-mini">📹 Kamera qo'shish</button>
          <button @click="addGate" :disabled="!isDrawn || isReadonly" class="action-btn-primary-mini">🚪 Eshik qo'shish</button>
          <button @click="addRoad" :disabled="!isDrawn || isReadonly" class="action-btn-primary-mini">🛣️ Yo'lak qo'shish</button>
        </div>
      </div>
    </div>

    <!-- ACTIONS (SAVE + CLEAR) dropdown -->
    <div class="dropdown-container">
      <button class="bottom-ctrl-btn" :class="{ active: showBottomSheet && bottomSheetTab === 'ACTIONS' }" @click="toggleBottomSheet('ACTIONS')">
        <span>⚡ Actions</span>
      </button>
      <div v-if="showBottomSheet && bottomSheetTab === 'ACTIONS'" class="dropdown-menu">
        <div class="dropdown-header">Actions</div>
        <div class="bs-pane">
          <div class="save-section-horizontal">
            <div class="action-item-mini">
              <p class="hint-text-mini">Barcha o'zgarishlarni tizimga saqlash.</p>
              <button class="save-btn-bs" @click="requestSave" :disabled="isReadonly">
                <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/>
                  <polyline points="17 21 17 13 7 13 7 21"/>
                </svg>
                <span>Saqlash</span>
              </button>
            </div>
            <div class="divider-v"></div>
            <div class="action-item-mini">
              <p class="hint-text-mini">Barcha elementlarni o'chirish.</p>
              <button class="clear-btn-bs" @click="clearAll" :disabled="isReadonly">
                <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <polyline points="3 6 5 6 21 6"></polyline>
                  <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path>
                </svg>
                <span>Tozalash</span>
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- SETTINGS dropdown -->
    <div class="dropdown-container">
      <button class="bottom-ctrl-btn" :class="{ active: showBottomSheet && bottomSheetTab === 'SETTINGS' }" @click="toggleBottomSheet('SETTINGS')">
        <span>⚙️ Sozlamalar</span>
      </button>
      <div v-if="showBottomSheet && bottomSheetTab === 'SETTINGS'" class="dropdown-menu">
        <div class="dropdown-header">Sozlamalar</div>
        <div class="dropdown-content-row">
          <div class="settings-section-horizontal">
            <div class="settings-item-mini">
              <label class="toggle-container-mini">
                <input type="checkbox" v-model="showMouseIndicators" style="display: none;">
                <div class="toggle-slider-mini"></div>
                <span class="toggle-label-mini">Sichqoncha ko'rsatkichi</span>
              </label>
            </div>
            <div class="settings-item-mini">
              <label class="toggle-container-mini">
                <input type="checkbox" v-model="isReadonly" @change="toggleReadonly" style="display: none;">
                <div class="toggle-slider-mini"></div>
                <span class="toggle-label-mini">Faqat o'qish rejimi</span>
              </label>
            </div>
          </div>
        </div>
      </div>
    </div>
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

  <div v-if="saveToastVisible" class="save-toast">Saqlandi ✓</div>
  <div class="zoom-controls" :style="{ right: isRightSidebarVisible ? '304px' : '24px' }">
    <button class="zoom-btn" @click="zoomBy(1 / 1.2)" title="Kichraytirish">−</button>
    <button class="zoom-level" @click="resetZoom" title="100% ga qaytarish">{{ zoomPercent }}%</button>
    <button class="zoom-btn" @click="zoomBy(1.2)" title="Kattalashtirish">+</button>
  </div>
  <div class="container" ref="containerRef" :style="{ left: '280px', width: isRightSidebarVisible ? 'calc(100vw - 560px)' : 'calc(100vw - 280px)', margin: '0', padding: '0', border: 'none', display: 'block' }">
    <Ruler 
      type="horizontal" 
      :offset="configKonva.x" 
      :scale="configKonva.scaleX" 
      :pixelsPerMeter="PIXELS_PER_METER" 
      :viewSize="configKonva.width"
      :leftOffset="0"
      :mousePos="currentMousePos.x"
      :showIndicator="showMouseIndicators"
    />
    <Ruler 
      type="vertical" 
      :offset="configKonva.y" 
      :scale="configKonva.scaleY" 
      :pixelsPerMeter="PIXELS_PER_METER" 
      :viewSize="configKonva.height"
      :leftOffset="0"
      :mousePos="currentMousePos.y"
      :showIndicator="showMouseIndicators"
    />
    <v-stage 
      :config="configKonva"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @wheel="handleWheel"
      @dragmove="handleStageDragMove"
      @dragend="handleStageDragEnd"
      :draggable="!isReadonly"
    >
      <v-layer>
        <!-- Fon to'rtburchagi - stage surish uchun -->
        <v-rect :config="{
          width: 100000,
          height: 100000,
          x: -50000,
          y: -50000,
          fill: '#c8d4e0',
          listening: true,
          name: 'background'
        }" />

        <ParkingArea
          :config="configRect"
          :slots="parkingSlots"
          :slotGroups="parkingSlotGroups"
          :slotAreas="parkingSlotAreas"
          :cameras="cameras"
          :selectedCameraId="selectedCameraId"
          :isReadonly="isReadonly"
          :gates="gates"
          :selectedGateId="selectedGateId"
          :isDrawn="isDrawn"
          :pixelsPerMeter="PIXELS_PER_METER"
          :textScale="configKonva.textScale"
          @dragmove="handleMouseMove"
          @delete-slot-area="handleDeleteSlotArea"
          @rotate-slot-area="handleRotateSlotArea"
          @edit-slot-area="data => handleEditSlotArea(data.id, false)"
          @update-slot-area-position="handleUpdateSlotAreaParams"
          @dragmove-slot-area="handleMouseMove"
          @slot-area-hover-change="handleSlotAreaHoverChange"
          v-if="isDrawn"
        />

        <!-- Yo'laklar -->
        <RoadArea
          v-for="road in roads"
          :key="road.id"
          :id="road.id"
          :x="road.x"
          :y="road.y"
          :width="road.width"
          :height="road.height"
          :rotation="road.rotation"
          :name="road.name"
          :pixelsPerMeter="PIXELS_PER_METER"
          :textScale="configKonva.textScale"
          :isSelected="road.id === selectedRoadId"
          :isReadonly="isReadonly"
          @select="id => handleSelectRoad(id, false)"
          @update-position="handleUpdateRoad"
          @dragmove="handleMouseMove"
        />

        <!-- Kameralar (Yo'laklar ustida ko'rinishi uchun keyinroq chiziladi) -->
        <Camera
          v-for="cam in cameras"
          :key="cam.id"
          :id="cam.id"
          :x="cam.x + configRect.x"
          :y="cam.y + configRect.y"
          :corners="cam.corners.map((v, i) => i % 2 === 0 ? v + configRect.x : v + configRect.y)"
          :textScale="configKonva.textScale"
          :parkingWidth="configRect.width"
          :parkingHeight="configRect.height"
          :pixelsPerMeter="PIXELS_PER_METER"
          :isSelected="cam.id === selectedCameraId"
          :isReadonly="isReadonly"
          @delete="handleDeleteCamera"
          @select="id => handleSelectCamera(id, false)"
          @update-body="handleUpdateCameraBody"
          @update-corner="handleUpdateCameraCorner"
          @dragmove="handleMouseMove"
        />

        <!-- Slot Area Tooltip (Barcha elementlar, jumladan yo'laklar ustida bo'lishi uchun) -->
        <v-group 
          v-if="hoverSlotArea" 
          :config="{ 
            x: hoverSlotArea.x + configRect.x, 
            y: hoverSlotArea.y + configRect.y,
            rotation: hoverSlotArea.rotation,
            listening: false 
          }"
        >
          <v-group :config="{ x: hoverSlotArea.width / 2, y: -25 / configKonva.textScale, listening: false }">
            <v-label :config="{ scaleX: configKonva.textScale, scaleY: configKonva.textScale, offsetX: 0, offsetY: 0, listening: false }">
              <v-tag :config="{
                fill: 'rgba(0,0,0,0.8)',
                cornerRadius: 4,
                stroke: '#00bfff',
                strokeWidth: 1 / configKonva.textScale,
                pointerDirection: 'down',
                pointerWidth: 10,
                pointerHeight: 6,
                lineJoin: 'round',
                shadowColor: 'black',
                shadowBlur: 10,
                shadowOpacity: 0.5
              }" />
              <v-text :config="{
                text: `${hoverSlotArea.slotCount} slots | W: ${(hoverSlotArea.width / hoverSlotArea.pixelsPerMeter).toFixed(2)}m | H: ${(hoverSlotArea.height / hoverSlotArea.pixelsPerMeter).toFixed(2)}m\nX: ${(hoverSlotArea.x / hoverSlotArea.pixelsPerMeter).toFixed(2)}m, Y: ${(hoverSlotArea.y / hoverSlotArea.pixelsPerMeter).toFixed(2)}m`,
                fontSize: 12,
                fill: '#ffffff',
                padding: 6,
                listening: false
              }" />
            </v-label>
          </v-group>
        </v-group>

        <!-- Eshiklar (Barcha elementlardan yuqorida bo'lishi uchun oxirida chiziladi) -->
    <Gate
      v-for="gate in gates"
      :key="gate.id"
      :id="gate.id"
      :side="gate.side"
      :x="gate.x + configRect.x"
      :y="gate.y + configRect.y"
      :width="gate.width"
      :type="gate.type"
      :isOpen="gate.isOpen"
      :rotation="gate.rotation"
      :textScale="configKonva.textScale"
      :parkingX="configRect.x"
      :parkingY="configRect.y"
      :parkingWidth="configRect.width"
      :parkingHeight="configRect.height"
      :pixelsPerMeter="PIXELS_PER_METER"
      :isSelected="gate.id === selectedGateId"
      :isReadonly="isReadonly"
      @delete="handleDeleteGate"
      @select="id => handleSelectGate(id, false)"
      @update-position="handleUpdateGatePosition"
      @update-gate="handlePanelUpdateGate"
      @dragmove="handleMouseMove"
    />


      </v-layer>
    </v-stage>
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
  background-color: #c8d4e0;
}

.container {
  position: fixed;
  top: 0;
  width: 100vw;
  height: 100vh;
  background-color: #c8d4e0;
  overflow: hidden;
}

/* ═══ LEFT SIDEBAR ═══ */
.left-sidebar {
  position: fixed;
  top: 0;
  left: 0;
  width: 280px;
  height: 100vh;
  background: #1a1a1a;
  border-right: 1px solid #333;
  box-shadow: 2px 0 12px rgba(0,0,0,0.5);
  z-index: 100;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* ═══ RIGHT SIDEBAR ═══ */
.right-sidebar {
  position: fixed;
  top: 0;
  right: 0;
  width: 280px;
  height: 100vh;
  background: #1a1a1a;
  border-left: 1px solid #333;
  box-shadow: -2px 0 12px rgba(0,0,0,0.5);
  z-index: 100;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.sidebar-header {
  padding: 12px 16px;
  background: #252525;
  border-bottom: 1px solid #333;
}

.sidebar-header.border-top {
  border-top: 1px solid #333;
}

.sidebar-title {
  margin: 0;
  font-size: 13px;
  color: #f5c518;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.element-browser {
  flex: 1;
  overflow-y: auto;
  background: #1a1a1a;
}

.element-list {
  display: flex;
  flex-direction: column;
}

.element-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 16px;
  cursor: pointer;
  border-bottom: 1px solid #222;
  transition: background 0.2s;
}

.element-item:hover {
  background: #252525;
}

.element-item.active {
  background: #2a2a2a;
  border-left: 3px solid #f5c518;
}

.element-icon {
  font-size: 16px;
}

.element-name {
  font-size: 13px;
  color: #e0e0e0;
}

.tab-pane {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  background: #1a1a1a;
}

/* Bottom Sheet */
.bottom-sheet {
  position: fixed;
  bottom: 0;
  left: 280px;
  right: 0;
  background: #1a1a1a;
  border-top: 1px solid #333;
  box-shadow: 0 -4px 20px rgba(0,0,0,0.5);
  z-index: 200;
  transform: translateY(100%);
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
  max-height: 250px;
}

.bottom-sheet.open {
  transform: translateY(0);
}

.bottom-sheet-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 16px;
  background: #252525;
  border-bottom: 1px solid #333;
}

.bottom-sheet-tabs {
  display: flex;
  gap: 16px;
}

.bs-tab-btn {
  padding: 12px 8px;
  background: transparent;
  border: none;
  border-bottom: 2px solid transparent;
  color: #888;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
}

.bs-tab-btn.active {
  color: #f5c518;
  border-bottom-color: #f5c518;
}

.close-bs-btn {
  background: transparent;
  border: none;
  color: #888;
  font-size: 24px;
  cursor: pointer;
  padding: 4px 8px;
}

.bottom-sheet-content {
  padding: 16px;
  overflow-y: auto;
}

.mode-tabs-horizontal {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
  overflow-x: auto;
  padding-bottom: 4px;
}

.mode-tab-mini {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: #2a2a2a;
  border: 1px solid #333;
  border-radius: 4px;
  color: #bbb;
  font-size: 12px;
  cursor: pointer;
  white-space: nowrap;
}

.mode-tab-mini.active {
  background: #f5c518;
  color: #111;
  border-color: #f5c518;
}

.form-section-horizontal {
  display: flex;
  flex-direction: column;
}

.form-row {
  display: flex;
  align-items: flex-end;
  gap: 12px;
  flex-wrap: wrap;
}

.form-label-mini {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.form-label-mini span {
  font-size: 11px;
  color: #888;
}

.form-input-mini, .form-select-mini {
  background: #2a2a2a;
  border: 1px solid #444;
  color: #fff;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 12px;
  width: 70px;
}

.form-select-mini {
  width: auto;
}

.action-btn-primary-mini {
  background: #f5c518;
  color: #111;
  border: none;
  border-radius: 4px;
  padding: 6px 12px;
  font-size: 12px;
  font-weight: 700;
  cursor: pointer;
}

.count-badge-mini {
  background: #333;
  color: #f5c518;
  padding: 4px 8px;
  border-radius: 10px;
  font-size: 11px;
}

.save-section-horizontal, .settings-section-horizontal {
  display: flex;
  align-items: center;
  gap: 20px;
}

.hint-text-mini {
  font-size: 12px;
  color: #888;
  margin: 0;
}

.save-btn-bs {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #f5c518;
  color: #111;
  border: none;
  border-radius: 4px;
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
}

.clear-btn-bs {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #e74c3c;
  color: #fff;
  border: none;
  border-radius: 4px;
  padding: 8px 16px;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
}

.clear-btn-bs:hover {
  background: #c0392b;
}

.clear-btn-bs:disabled {
  background: #555;
  color: #888;
  cursor: not-allowed;
  opacity: 0.6;
}

.action-item-mini {
  display: flex;
  flex-direction: column;
  gap: 8px;
  align-items: center;
  flex: 1;
}

.divider-v {
  width: 1px;
  height: 60px;
  background: #333;
  margin: 0 10px;
}

.settings-item-mini {
  display: flex;
  align-items: center;
}

.toggle-container-mini {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}

.toggle-label-mini {
  font-size: 12px;
  color: #eee;
}

.toggle-slider-mini {
  width: 34px;
  height: 18px;
  background: #333;
  border-radius: 9px;
  position: relative;
  transition: 0.3s;
}

.toggle-slider-mini::before {
  content: "";
  position: absolute;
  width: 14px;
  height: 14px;
  background: #888;
  border-radius: 50%;
  top: 2px;
  left: 2px;
  transition: 0.3s;
}

input:checked + .toggle-slider-mini {
  background: #f5c518;
}

input:checked + .toggle-slider-mini::before {
  transform: translateX(16px);
  background: #111;
}

/* Bottom Controls */
.bottom-controls {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 12px;
  background: rgba(26, 26, 26, 0.9);
  backdrop-filter: blur(10px);
  padding: 8px 16px;
  border-radius: 30px;
  border: 1px solid #444;
  z-index: 200;
  box-shadow: 0 4px 15px rgba(0,0,0,0.3);
}

.dropdown-container {
  position: relative;
}

.dropdown-menu {
  position: absolute;
  bottom: calc(100% + 12px);
  left: 50%;
  transform: translateX(-50%);
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 12px;
  padding: 16px;
  width: max-content;
  min-width: 200px;
  max-width: 90vw;
  box-shadow: 0 10px 25px rgba(0,0,0,0.5);
  z-index: 210;
  animation: slideUp 0.2s ease-out;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.dropdown-content-row {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 12px;
}

@keyframes slideUp {
  from { opacity: 0; transform: translate(-50%, 10px); }
  to { opacity: 1; transform: translate(-50%, 0); }
}

.dropdown-header {
  font-size: 12px;
  font-weight: 700;
  color: #f5c518;
  text-transform: uppercase;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid #333;
}

.bottom-ctrl-btn {
  background: transparent;
  border: none;
  color: #bbb;
  font-size: 13px;
  font-weight: 600;
  padding: 6px 12px;
  cursor: pointer;
  border-radius: 20px;
  transition: all 0.2s;
}

.bottom-ctrl-btn:hover {
  background: rgba(255,255,255,0.1);
  color: #fff;
}

.bottom-ctrl-btn.active {
  background: #f5c518;
  color: #111;
}

.mode-tab {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 10px 6px;
  cursor: pointer;
  border: 1px solid #333;
  background: #2a2a2a;
  color: #888;
  font-size: 11px;
  font-weight: 600;
  transition: all 0.2s;
}

.mode-tab:not(:first-child) {
  border-left: none;
}

.mode-tab:first-child {
  border-radius: 6px 0 0 6px;
}

.mode-tab:last-child {
  border-radius: 0 6px 6px 0;
}

.mode-tab.active {
  background: #f5c518;
  color: #111;
  border-color: #f5c518;
}

.mode-tab:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  filter: grayscale(1);
}

.tab-icon {
  font-size: 20px;
  line-height: 1;
}

.tab-label {
  font-size: 10px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* Save Section */
.save-section {
  padding: 16px;
  background: #1e1e1e;
  height: 100%;
}

.save-btn-sidebar {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 10px 16px;
  background: linear-gradient(135deg, #4caf50 0%, #45a049 100%);
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 700;
  transition: all 0.2s;
  box-shadow: 0 2px 8px rgba(76, 175, 80, 0.3);
}

.save-btn-sidebar:hover {
  background: linear-gradient(135deg, #45a049 0%, #3d8b40 100%);
  box-shadow: 0 4px 12px rgba(76, 175, 80, 0.5);
  transform: translateY(-1px);
}

.save-btn-sidebar:active {
  transform: translateY(0);
}

/* Settings Section */
.settings-section-horizontal {
  display: flex;
  flex-direction: column;
  gap: 12px;
  align-items: stretch;
}

.settings-item-mini {
  display: flex;
  align-items: center;
}

.toggle-container-mini {
  display: flex;
  align-items: center;
  gap: 12px;
  cursor: pointer;
  user-select: none;
}

.toggle-label-mini {
  font-size: 13px;
  color: #eee;
  font-weight: 500;
}

.toggle-slider-mini {
  width: 40px;
  height: 20px;
  background: #333;
  border-radius: 20px;
  position: relative;
  transition: all 0.3s ease;
  box-shadow: inset 0 1px 3px rgba(0,0,0,0.4);
}

.toggle-slider-mini::before {
  content: "";
  position: absolute;
  width: 16px;
  height: 16px;
  background: #fff;
  border-radius: 50%;
  top: 2px;
  left: 2px;
  transition: all 0.3s cubic-bezier(0.68, -0.55, 2.65, 1.55);
}

input:checked + .toggle-slider-mini {
  background: #f5c518;
}

input:checked + .toggle-slider-mini::before {
  transform: translateX(20px);
  background: #111;
}

/* Dropdown forms horizontal layout */
.add-forms-container {
  margin-top: 8px;
}

.form-section-horizontal {
  display: flex;
  align-items: center;
}

.form-row {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 8px;
  flex-wrap: nowrap;
}

.action-btn-primary-mini {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 8px 16px;
  background: #2a2a2a;
  border: 1px solid #333;
  border-radius: 6px;
  color: #eee;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.action-btn-primary-mini:hover:not(:disabled) {
  background: #333;
  border-color: #f5c518;
  color: #f5c518;
}

.action-btn-primary-mini:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  background: #1a1a1a;
  color: #666;
  border-color: #222;
}

.save-btn-bs:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  background: #222;
  color: #666;
  filter: grayscale(1);
}

.settings-item {
  margin-bottom: 20px;
}

.toggle-container {
  display: flex;
  align-items: center;
  gap: 12px;
  cursor: pointer;
  user-select: none;
}

.toggle-label {
  font-size: 13px;
  font-weight: 600;
  color: #e8e8e8;
}

/* Custom Checkbox Toggle */
.toggle-slider {
  position: relative;
  width: 36px;
  height: 20px;
  background-color: #333;
  border-radius: 20px;
  transition: .3s;
}

.toggle-slider:before {
  position: absolute;
  content: "";
  height: 14px;
  width: 14px;
  left: 3px;
  bottom: 3px;
  background-color: white;
  border-radius: 50%;
  transition: .3s;
}

input:checked + .toggle-slider {
  background-color: #4caf50;
}

input:checked + .toggle-slider:before {
  transform: translateX(16px);
}

.toggle-container input {
  display: none;
}

/* Sidebar Content (Forms) */
.sidebar-content {
  flex: 1;
  padding: 16px;
}

.form-section {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.section-title {
  font-size: 13px;
  font-weight: 700;
  color: #f5c518;
  text-transform: uppercase;
  letter-spacing: 0.8px;
  margin: 0 0 8px 0;
  padding-bottom: 8px;
  border-bottom: 1px solid #2e2e2e;
}

.form-label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 12px;
  color: #999;
}

.form-label span {
  font-weight: 600;
  color: #aaa;
}

.form-input, .form-select {
  width: 100%;
  padding: 8px 10px;
  background: #2a2a2a;
  border: 1px solid #3a3a3a;
  color: #e8e8e8;
  border-radius: 4px;
  font-size: 13px;
  transition: all 0.2s;
}

.form-input:focus, .form-select:focus {
  outline: none;
  border-color: #f5c518;
  background: #2f2f2f;
}

.action-btn-primary {
  width: 100%;
  padding: 10px 16px;
  background: #f5c518;
  color: #111;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: 700;
  font-size: 13px;
  transition: all 0.2s;
  margin-top: 4px;
}

.action-btn-primary:hover {
  background: #ffd84d;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(245, 197, 24, 0.3);
}

.action-btn-primary:active {
  transform: translateY(0);
}

.info-text {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 10px;
  background: #222;
  border-radius: 5px;
  border: 1px solid #333;
}

.count-badge {
  display: inline-block;
  padding: 4px 10px;
  background: #2a2a2a;
  color: #f5c518;
  border-radius: 12px;
  font-size: 11px;
  font-weight: 700;
  text-align: center;
}

.hint-text {
  margin: 0;
  font-size: 11px;
  color: #777;
  font-style: italic;
}

/* Properties Section */
.properties-section {
  padding: 0;
}

.divider {
  height: 1px;
  background: linear-gradient(90deg, transparent, #444, transparent);
  margin: 8px 0;
}

.camera-btn {
  background: #00bfff;
  color: #111;
}

.camera-btn:hover {
  background: #33ccff;
}

.gate-btn {
  background: #4caf50;
  color: #111;
}

.gate-btn:hover {
  background: #66bb6a;
}

.gate-select {
  background: #2a2a2a;
  border: 1px solid #444;
  color: #e0e0e0;
  border-radius: 4px;
  padding: 3px 6px;
  font-size: 12px;
  cursor: pointer;
}

.hint {
  margin: 0;
  font-size: 11px;
  color: #666;
  font-style: italic;
}

.cam-count {
  font-size: 12px;
  color: #888;
}

.cancel-btn {
  margin-top: 4px;
  padding: 6px 10px;
  background: #c0392b;
  color: white;
  border: none;
  cursor: pointer;
  border-radius: 4px;
  font-size: 13px;
}

.cancel-btn:hover {
  background: #e74c3c;
}

.zoom-controls {
  position: fixed;
  bottom: 24px;
  right: 24px;
  z-index: 100;
  display: flex;
  align-items: center;
  gap: 2px;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  box-shadow: 0 4px 16px rgba(0,0,0,0.5);
  padding: 4px;
  transition: right 0.3s ease;
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
  color: #ccc;
  display: flex;
  align-items: center;
  justify-content: center;
}

.zoom-btn:hover {
  background: #2a2a2a;
  color: #fff;
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
  color: #f5c518;
}

.zoom-level:hover {
  background: #2a2a2a;
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
  background: #1a1a1a;
  color: #f5c518;
  border: 1px solid #f5c518;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(0,0,0,0.4);
}

.save-btn:hover {
  background: #f5c518;
  color: #111;
}

.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.6);
  z-index: 200;
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal {
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 8px;
  padding: 24px 28px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.6);
  min-width: 240px;
  text-align: center;
}

.modal p {
  margin: 0 0 20px;
  font-size: 15px;
  color: #ccc;
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
  background: #2a2a2a;
  color: #aaa;
  border: 1px solid #444;
}

.modal-btn--cancel:hover {
  background: #333;
}

.modal-btn--confirm {
  background: #f5c518;
  color: #111;
  font-weight: 700;
}

.modal-btn--confirm:hover {
  background: #ffd84d;
}

.save-toast {
  position: fixed;
  bottom: 72px;
  right: 24px;
  z-index: 300;
  background: #1a1a1a;
  border: 1px solid #f5c518;
  color: #f5c518;
  font-size: 13px;
  padding: 8px 16px;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.4);
  pointer-events: none;
}
</style>
