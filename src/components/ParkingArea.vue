<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'
import ParkingSlot from './ParkingSlot.vue'
import ParkingSlotGroup from './ParkingSlotGroup.vue'
import SlotArea from './SlotArea.vue'
import Camera from './Camera.vue'
import Gate from './Gate.vue'

interface Slot {
  id: string
  x: number
  y: number
  width: number
  height: number
}

interface SlotGroup {
  id: string
  name: string
  x: number
  y: number
  width: number
  height: number
  slots: Slot[]
}

const props = defineProps<{
  config: {
    x: number
    y: number
    width: number
    height: number
    fill?: string
    opacity?: number
  }
  slots: Slot[]
  slotGroups: SlotGroup[]
  slotAreas: any[]
  cameras: any[]
  selectedCameraId: string | null
  isReadonly?: boolean
  isDrawn: boolean
  pixelsPerMeter: number
  textScale: number
}>()

const emit = defineEmits([
  'transform', 'dragmove', 'click', 
  'delete-slot-area', 'rotate-slot-area', 'edit-slot-area', 'update-slot-area-position', 
  'dragmove-slot-area', 'slot-area-hover-change'
])

const isHovered = ref(false)

const areaConfig = computed(() => ({
  ...props.config,
  draggable: false,
  listening: true,
  name: 'mainRect',
  stroke: '#f5c518',
  strokeWidth: 4
}))

const handleClick = (e: any) => emit('click', e)

</script>

<template>
  <v-group>
    <!-- Asosiy parking maydoni -->
    <v-rect 
      :config="areaConfig"
      @mouseenter="isHovered = true"
      @mouseleave="isHovered = false"
    />

    <!-- Parking ichidagi elementlar -->
    <v-group :config="{ x: config.x, y: config.y }">
      <!-- Alohida slotlar -->
      <ParkingSlot 
        v-for="slot in slots" 
        :key="slot.id" 
        :config="slot" 
      />

      <!-- Slot guruhlari -->
      <ParkingSlotGroup 
        v-for="group in slotGroups" 
        :key="group.id" 
        :config="{ x: group.x, y: group.y, width: group.width, height: group.height }"
        :slots="group.slots"
        :name="group.name"
        :textScale="textScale"
      />

      <!-- Slot Area lar -->
      <SlotArea
        v-for="area in slotAreas"
        :key="area.id"
        :id="area.id"
        :config="area.config"
        :slotCount="area.slotCount"
        :slotWidth="area.slotWidth"
        :slotHeight="area.slotHeight"
        :slotAngle="area.slotAngle"
        :pixelsPerMeter="pixelsPerMeter"
        :textScale="textScale"
        :parkingWidth="config.width"
        :parkingHeight="config.height"
        :isReadonly="isReadonly"
        @edit="emit('edit-slot-area', $event)"
        @update-position="(pos) => emit('update-slot-area-position', { id: area.id, ...pos })"
        @dragmove="emit('dragmove-slot-area', $event)"
        @hover-change="emit('slot-area-hover-change', $event)"
      />
    </v-group>
  </v-group>
</template>
