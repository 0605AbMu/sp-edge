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
  width: 0,
  height: 0,
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
  width: 0,
  height: 0,
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
    if (!raw) return
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
      configKonva.value.x = Number(data.configKonva.x) ?? 280
      configKonva.value.y = Number(data.configKonva.y) ?? 0
      configKonva.value.scaleX = Number(data.configKonva.scaleX) || 1
      configKonva.value.scaleY = Number(data.configKonva.scaleY) || 1
      configKonva.value.textScale = 1 / (Number(data.configKonva.scaleX) || 1)
    } else {
      configKonva.value.x = 280
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

// Slot Area parametrlari
const slotParams = ref({
  count: 5,
  width: 2.5,
  height: 5,
  angle: 0
})

const parkingSlotAreas = shallowRef<any[]>([])
const selectedSlotAreaId = ref<string | null>(null)
// editingSlotAreaId va editPopupPos olib tashlandi - endi PropertiesPanel sidebar ichida

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

const handlePanelUpdateCamera = (data: any) => {
  handleUpdateCameraBody(data)
}

const handlePanelUpdateSlotArea = (data: any) => {
  const area = parkingSlotAreas.value.find(a => a.id === data.id)
  if (!area) return
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
}

const handlePanelUpdateGate = (data: any) => {
  const gate = gates.value.find(g => g.id === data.id)
  if (!gate) return
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
}

const mode = ref<'PARKING' | 'SLOT_AREA' | 'CAMERA' | 'GATE' | 'ROAD'>('PARKING')
const hoverSlotArea = ref<any>(null)

const handleSlotAreaHoverChange = (data: any) => {
  if (data.isHovered) {
    hoverSlotArea.value = data
  } else if (hoverSlotArea.value?.id === data.id) {
    hoverSlotArea.value = null
  }
}

// --- Yo'lak (Road) state ---
interface RoadData {
  id: string
  x: number
  y: number
  width: number  // meters
  height: number // meters
  rotation: number
  name: string
}

const roads = shallowRef<RoadData[]>([])
const selectedRoadId = ref<string | null>(null)

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

const handleSelectRoad = (id: string) => {
  selectedCameraId.value = null
  selectedSlotAreaId.value = null
  selectedGateId.value = null
  selectedRoadId.value = id
  sidebarTab.value = 'EDIT' // isReadonly dan qat'iy nazar EDIT tabiga o'tamiz
}

const handleUpdateRoad = (data: Partial<RoadData> & { id: string }) => {
  const road = roads.value.find(r => r.id === data.id)
  if (road) {
    if (data.x !== undefined) road.x = data.x
    if (data.y !== undefined) road.y = data.y
    if (data.width !== undefined) road.width = data.width
    if (data.height !== undefined) road.height = data.height
    if (data.rotation !== undefined) road.rotation = data.rotation
    if (data.name !== undefined) road.name = data.name
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
interface CameraData {
  id: string
  x: number       // parking-relative pixels
  y: number
  corners: number[] // [x1,y1, x2,y2, x3,y3, x4,y4]
}

const cameras = shallowRef<CameraData[]>([])
const selectedCameraId = ref<string | null>(null)
const editingCameraId = ref<string | null>(null)
const isReadonly = ref(true)

const toggleReadonly = () => {
  // Avvalgi mantiq tanlovlarni o'chirib yuborar edi, endi buni olib tashlaymiz
  // Chunki element tanlangan holda ham readonly ga o'tish mumkin bo'lishi kerak
}

// --- Eshik (Gate) state ---
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

const gates = shallowRef<GateData[]>([])
const selectedGateId = ref<string | null>(null)

const gateParams = ref({
  type: 'ENTRANCE' as 'ENTRANCE' | 'EXIT' | 'BOTH',
  side: 'TOP' as 'TOP' | 'BOTTOM' | 'LEFT' | 'RIGHT',
  width: 3
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

const handleSelectGate = (id: string) => {
  selectedCameraId.value = null
  selectedSlotAreaId.value = null
  selectedRoadId.value = null
  selectedGateId.value = id
  sidebarTab.value = 'EDIT' // isReadonly dan qat'iy nazar EDIT tabiga o'tamiz
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
  const gate = gates.value.find(g => g.id === data.id)
  if (gate) {
    // Gate.vue provides coordinates relative to the same parent as configRect
    // but the Gate component in template uses x + configRect.x
    // so we store coordinates relative to ParkingArea (0..W, 0..H)
    gate.x = data.x - configRect.value.x
    gate.y = data.y - configRect.value.y
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

const handleSelectCamera = (id: string) => {
  // Readonly rejimida bo'lganda ham tanlash mumkin
  selectedSlotAreaId.value = null
  selectedGateId.value = null
  selectedRoadId.value = null
  selectedCameraId.value = id
  sidebarTab.value = 'EDIT' // Har doim EDIT tabiga o'tamiz
}

const handleUpdateCameraBody = (data: { id: string; x: number; y: number; corners?: number[] }) => {
  const cam = cameras.value.find(c => c.id === data.id)
  if (cam) {
    cam.x = data.x - configRect.value.x
    cam.y = data.y - configRect.value.y
    if (data.corners) {
      cam.corners = data.corners.map((v, i) => i % 2 === 0 ? v - configRect.value.x : v - configRect.value.y)
    }
  }
}

const handleUpdateCameraCorner = (data: { id: string; index: number; x: number; y: number }) => {
  const cam = cameras.value.find(c => c.id === data.id)
  if (cam) {
    cam.corners[data.index * 2] = data.x - configRect.value.x
    cam.corners[data.index * 2 + 1] = data.y - configRect.value.y
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
    cameras.value.push(newCam)
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
  
  parkingSlotAreas.value.push(newArea)
  // Yangi SlotArea qo'shilganda avtomatik tanlab, sidebar'da ko'rsatamiz
  handleEditSlotArea({ id, x: 50, y: 50 })
}

const handleDeleteSlotArea = (id: string) => {
  parkingSlotAreas.value = [...parkingSlotAreas.value.filter(a => a.id !== id)]
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

const handleCloneSlotArea = (id: string) => {
  if (isReadonly.value) return
  const area = parkingSlotAreas.value.find(a => a.id === id)
  if (area) {
    const newArea = JSON.parse(JSON.stringify(area))
    newArea.id = `slot-area-${Date.now()}`
    newArea.config.x += PIXELS_PER_METER
    newArea.config.y += PIXELS_PER_METER
    parkingSlotAreas.value.push(newArea)
    selectedSlotAreaId.value = newArea.id
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
    const sh = (area.slotHeight || 0) * (area.pixelsPerMeter || 50)
    const angleRad = ((area.slotAngle || 0) * Math.PI) / 180
    const sOffset = sh * Math.tan(angleRad)
    
    area.config.width = (area.slotCount || 0) * sw + Math.abs(sOffset)
    area.config.height = sh
  }
}

const handleEditSlotArea = (data: { id: string, x: number, y: number, screenX?: number, screenY?: number }) => {
  // Slot areani tanlaymiz - sidebar properties panelda ko'rsatiladi
  selectedCameraId.value = null
  selectedGateId.value = null
  selectedRoadId.value = null
  selectedSlotAreaId.value = data.id
  sidebarTab.value = 'EDIT' // Har doim EDIT tabiga o'tamiz
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
  selectedSlotAreaId.value = null
  cameras.value = []
  selectedCameraId.value = null
  gates.value = []
  selectedGateId.value = null
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

    // Kamera yoki uning burchaklaridan boshqa joyga bosilsa — deaktiv
    if (!isCameraElement) {
      selectedCameraId.value = null
    }

    // Gate dan boshqa joyga bosilsa — deaktiv
    const isGateElement = targetName === 'gate'
    if (!isGateElement) {
      selectedGateId.value = null
    }

    if (isStageClicked) {
      selectedShapeName.value = ''
      selectedSlotAreaId.value = null
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
      } else {
        selectedShapeName.value = ''
        selectedSlotAreaId.value = null
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
  configKonva.value.x = 280 // Sidebar width
  configKonva.value.y = 0
}

const sidebarTab = ref('EDIT')
</script>

<template>
  <!-- Chap sidebar - barcha boshqaruvlar -->
  <div class="left-sidebar">
    <!-- Main sidebar tabs -->
    <div class="sidebar-tabs">
      <button :class="{ active: sidebarTab === 'ADD' }" @click="sidebarTab = 'ADD'" class="sidebar-tab-btn">
        <span class="tab-icon-small">➕</span>
      </button>
      <button :class="{ active: sidebarTab === 'EDIT' }" @click="sidebarTab = 'EDIT'" class="sidebar-tab-btn">
        <span class="tab-icon-small">⚙️</span>
      </button>
      <button :class="{ active: sidebarTab === 'SAVE' }" @click="sidebarTab = 'SAVE'" class="sidebar-tab-btn">
        <span class="tab-icon-small">💾</span>
      </button>
      <button :class="{ active: sidebarTab === 'SETTINGS' }" @click="sidebarTab = 'SETTINGS'" class="sidebar-tab-btn">
        <span class="tab-icon-small">⚙️</span>
      </button>
    </div>

    <div v-show="sidebarTab === 'ADD'" class="tab-pane">
      <!-- Mode selector tabs -->
      <div class="mode-tabs">
        <button :class="{ active: mode === 'PARKING' }" @click="mode = 'PARKING'" :disabled="isReadonly" class="mode-tab">
          <span class="tab-icon">🅿️</span>
        </button>
        <button :class="{ active: mode === 'SLOT_AREA' }" @click="mode = 'SLOT_AREA'" :disabled="!isDrawn || isReadonly" class="mode-tab">
          <span class="tab-icon">🚗</span>
        </button>
        <button :class="{ active: mode === 'CAMERA' }" @click="mode = 'CAMERA'" :disabled="!isDrawn || isReadonly" class="mode-tab">
          <span class="tab-icon">📹</span>
        </button>
        <button :class="{ active: mode === 'GATE' }" @click="mode = 'GATE'" :disabled="!isDrawn || isReadonly" class="mode-tab">
          <span class="tab-icon">🚪</span>
        </button>
        <button :class="{ active: mode === 'ROAD' }" @click="mode = 'ROAD'" :disabled="!isDrawn || isReadonly" class="mode-tab">
          <span class="tab-icon">🛣️</span>
        </button>
      </div>

      <!-- Qo'shish formlari -->
      <div class="sidebar-content">
        <div v-show="mode === 'PARKING'" class="form-section">
          <h3 class="section-title">Parking maydon</h3>
          <label class="form-label">
            <span>Kenglik (m)</span>
            <input type="number" v-model.number="parkingParams.width" :disabled="isReadonly" step="0.1" @input="updateParkingDimensions" class="form-input" />
          </label>
          <label class="form-label">
            <span>Uzunlik (m)</span>
            <input type="number" v-model.number="parkingParams.height" :disabled="isReadonly" step="0.1" @input="updateParkingDimensions" class="form-input" />
          </label>
          <button v-if="!isDrawn" @click="createParking" :disabled="isReadonly" class="action-btn-primary">Chizish</button>
        </div>

        <div v-show="mode === 'SLOT_AREA'" class="form-section">
          <h3 class="section-title">Yangi slot maydon</h3>
          <label class="form-label">
            <span>Slot soni</span>
            <input type="number" v-model.number="slotParams.count" :disabled="isReadonly" class="form-input" />
          </label>
          <label class="form-label">
            <span>Kenglik (m)</span>
            <input type="number" v-model.number="slotParams.width" :disabled="isReadonly" step="0.1" class="form-input" />
          </label>
          <label class="form-label">
            <span>Uzunlik (m)</span>
            <input type="number" v-model.number="slotParams.height" :disabled="isReadonly" step="0.1" class="form-input" />
          </label>
          <label class="form-label">
            <span>Burchak (°)</span>
            <input type="number" v-model.number="slotParams.angle" :disabled="isReadonly" class="form-input" />
          </label>
          <button @click="addSlotArea" :disabled="isReadonly" class="action-btn-primary">+ Qo'shish</button>
        </div>

        <div v-show="mode === 'CAMERA'" class="form-section">
          <h3 class="section-title">Kameralar</h3>
          <button @click="addCamera" :disabled="isReadonly" class="action-btn-primary">+ Kamera qo'shish</button>
          <div v-if="cameras.length > 0" class="info-text">
            <span class="count-badge">{{ cameras.length }} ta</span>
            <p class="hint-text">Kamera ustiga bosib aktiv qiling</p>
          </div>
        </div>

        <div v-show="mode === 'GATE'" class="form-section">
          <h3 class="section-title">Yangi eshik</h3>
          <label class="form-label">
            <span>Turi</span>
            <select v-model="gateParams.type" :disabled="isReadonly" class="form-select">
              <option value="ENTRANCE">Kirish</option>
              <option value="EXIT">Chiqish</option>
              <option value="BOTH">Kirish/Chiqish</option>
            </select>
          </label>
          <label class="form-label">
            <span>Tomon</span>
            <select v-model="gateParams.side" :disabled="isReadonly" class="form-select">
              <option value="TOP">Yuqori</option>
              <option value="BOTTOM">Pastki</option>
              <option value="LEFT">Chap</option>
              <option value="RIGHT">O'ng</option>
            </select>
          </label>
          <label class="form-label">
            <span>Kenglik (m)</span>
            <input type="number" v-model.number="gateParams.width" :disabled="isReadonly" step="0.5" min="1" class="form-input" />
          </label>
          <button @click="addGate" :disabled="isReadonly" class="action-btn-primary">+ Eshik qo'shish</button>
          <div v-if="gates.length > 0" class="info-text">
            <span class="count-badge">{{ gates.length }} ta</span>
          </div>
        </div>

        <div v-show="mode === 'ROAD'" class="form-section">
          <h3 class="section-title">Yo'lakchalar</h3>
          <p class="section-desc">Mashinalar harakatlanishi uchun yo'l chizig'i</p>
          <button @click="addRoad" :disabled="isReadonly" class="action-btn-primary">+ Yo'lak qo'shish</button>
          <div v-if="roads.length > 0" class="info-text">
            <span class="count-badge">{{ roads.length }} ta</span>
          </div>
        </div>
      </div>
    </div>

    <div v-show="sidebarTab === 'EDIT'" class="tab-pane">
      <!-- Properties Panel - har doim render qilinadi, lekin v-show bilan ko'rsatiladi -->
      <div v-show="selectedCamera || selectedSlotArea || selectedGate || selectedRoad" class="properties-section">
        <PropertiesPanel
          :camera="selectedCamera"
          :slotArea="selectedSlotArea"
          :gate="selectedGate"
          :road="selectedRoad"
          :pixelsPerMeter="PIXELS_PER_METER"
          :isReadonly="isReadonly"
          @update-camera="handlePanelUpdateCamera"
          @update-slot-area="handlePanelUpdateSlotArea"
          @update-gate="handlePanelUpdateGate"
          @update-road="handleUpdateRoad"
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
      <div v-show="!(selectedCamera || selectedSlotArea || selectedGate || selectedRoad)" class="empty-state">
        <p>Tahrirlash uchun elementni tanlang</p>
      </div>
    </div>

    <div v-show="sidebarTab === 'SAVE'" class="tab-pane">
      <!-- Saqlash tugmasi -->
      <div class="save-section">
        <h3 class="section-title">Saqlash</h3>
        <p class="hint-text">Barcha o'zgarishlarni tizimga saqlash uchun quyidagi tugmani bosing.</p>
        <button class="save-btn-sidebar" @click="requestSave" title="Saqlash (Ctrl+S)">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/>
            <polyline points="17 21 17 13 7 13 7 21"/>
            <polyline points="7 3 7 8 15 8"/>
          </svg>
          <span>Saqlash</span>
        </button>
      </div>
    </div>

    <div v-show="sidebarTab === 'SETTINGS'" class="tab-pane">
      <!-- Sozlamalar -->
      <div class="settings-section">
        <h3 class="section-title">Sozlamalar</h3>
        
        <div class="settings-item">
          <label class="toggle-container">
            <input type="checkbox" v-model="showMouseIndicators">
            <span class="toggle-slider"></span>
            <span class="toggle-label">Sichqoncha ko'rsatkichi</span>
          </label>
          <p class="hint-text">Chizg'ichda sichqoncha koordinatalarini ko'rsatish.</p>
        </div>

        <div class="settings-item">
          <label class="toggle-container">
            <input type="checkbox" v-model="isReadonly" @change="toggleReadonly">
            <span class="toggle-slider"></span>
            <span class="toggle-label">Faqat o'qish rejimi</span>
          </label>
          <p class="hint-text">Yoqilganda barcha elementlar bloklanadi, faqat kameralarni ko'rish mumkin.</p>
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

  <!-- Saqlandi xabari -->
  <div v-if="saveToastVisible" class="save-toast">Saqlandi ✓</div>
  <div class="zoom-controls">
    <button class="zoom-btn" @click="zoomBy(1 / 1.2)" title="Kichraytirish">−</button>
    <button class="zoom-level" @click="resetZoom" title="100% ga qaytarish">{{ zoomPercent }}%</button>
    <button class="zoom-btn" @click="zoomBy(1.2)" title="Kattalashtirish">+</button>
  </div>
  <div class="container" ref="containerRef">
    <Ruler 
      type="horizontal" 
      :offset="configKonva.x - 280" 
      :scale="configKonva.scaleX" 
      :pixelsPerMeter="PIXELS_PER_METER" 
      :viewSize="configKonva.width - 280"
      :leftOffset="280"
      :mousePos="currentMousePos.x"
      :showIndicator="showMouseIndicators"
    />
    <Ruler 
      type="vertical" 
      :offset="configKonva.y - 25" 
      :scale="configKonva.scaleY" 
      :pixelsPerMeter="PIXELS_PER_METER" 
      :viewSize="configKonva.height - 25"
      :leftOffset="280"
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
          @edit-slot-area="handleEditSlotArea"
          @update-slot-area-position="handleUpdateSlotAreaParams"
          @dragmove-slot-area="handleMouseMove"
          @slot-area-hover-change="handleSlotAreaHoverChange"
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
          @select="handleSelectRoad"
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
          @select="handleSelectCamera"
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
      @select="handleSelectGate"
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
  position: relative;
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
  overflow-y: auto;
  overflow-x: hidden;
}

.sidebar-tabs {
  display: flex;
  background: #252525;
  border-bottom: 1px solid #333;
  overflow-x: auto;
  scrollbar-width: none; /* Firefox */
  -ms-overflow-style: none; /* IE/Edge */
}

.sidebar-tabs::-webkit-scrollbar {
  display: none; /* Chrome, Safari, Opera */
}

.sidebar-tab-btn {
  flex: 1;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0;
  padding: 12px 16px;
  background: transparent;
  border: none;
  border-bottom: 2px solid transparent;
  color: #888;
  font-size: 11px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
}

.sidebar-tab-btn:hover {
  background: rgba(255,255,255,0.05);
  color: #bbb;
}

.sidebar-tab-btn.active {
  color: #f5c518;
  border-bottom-color: #f5c518;
  background: rgba(245, 197, 24, 0.05);
}

.sidebar-tab-btn.disabled {
  opacity: 0.5;
  cursor: not-allowed;
  filter: grayscale(1);
}

.sidebar-tab-btn.disabled:hover {
  background: transparent;
  color: #888;
}

.tab-icon-small {
  font-size: 14px;
}

.tab-pane {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.empty-state {
  padding: 40px 20px;
  text-align: center;
  color: #666;
  font-style: italic;
  font-size: 13px;
}

.left-sidebar::-webkit-scrollbar {
  width: 6px;
}
.left-sidebar::-webkit-scrollbar-track {
  background: #111;
}
.left-sidebar::-webkit-scrollbar-thumb {
  background: #444;
  border-radius: 3px;
}

/* Mode Tabs */
.mode-tabs {
  display: flex;
  gap: 0;
  padding: 12px;
  background: #212121;
  border-bottom: 1px solid #333;
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
.settings-section {
  padding: 16px;
  background: #1e1e1e;
  height: 100%;
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
