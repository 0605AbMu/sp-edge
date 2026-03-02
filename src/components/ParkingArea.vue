<script setup lang="ts">
import { computed, ref, onMounted } from 'vue'
import ParkingSlot from './ParkingSlot.vue'
import ParkingSlotGroup from './ParkingSlotGroup.vue'
import SlotArea from './SlotArea.vue'

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
  isDrawn: boolean
  pixelsPerMeter: number
  textScale: number
}>()

const emit = defineEmits(['transform', 'dragmove', 'mouseenter', 'mouseleave', 'click', 'delete-slot-area', 'rotate-slot-area', 'edit-slot-area', 'update-slot-area-position'])

const isHovered = ref(false)

const areaConfig = computed(() => ({
  ...props.config,
  draggable: false,
  listening: true,
  name: 'mainRect'
}))

const handleMouseEnter = (e: any) => {
  isHovered.value = true
  emit('mouseenter', e)
}
const handleMouseLeave = (e: any) => {
  isHovered.value = false
  emit('mouseleave', e)
}
const handleClick = (e: any) => emit('click', e)

const tooltipText = computed(() => {
  const w_m = (props.config.width / props.pixelsPerMeter).toFixed(2)
  const h_m = (props.config.height / props.pixelsPerMeter).toFixed(2)
  return `w: ${w_m}m, h: ${h_m}m`
})
</script>

<template>
  <v-group>
    <!-- Asosiy parking maydoni -->
    <v-rect 
      :config="areaConfig"
      @mouseenter="handleMouseEnter"
      @mouseleave="handleMouseLeave"
    />

    <!-- Parking Tooltip (Hoverda o'lchamlarni ko'rsatish) -->
    <v-label v-if="isHovered && isDrawn" :config="{ 
      x: config.x, 
      y: config.y - 5, 
      scaleX: textScale, 
      scaleY: textScale 
    }">
      <v-tag :config="{ fill: 'black', opacity: 0.7, pointerDirection: 'bottom', pointerWidth: 10, pointerHeight: 10, lineJoin: 'round', cornerRadius: 5 }" />
      <v-text :config="{ text: tooltipText, fontSize: 12, fill: 'white', padding: 5 }" />
    </v-label>

    <!-- Parking ichidagi elementlar (faqat maydon chizib bo'lingandan keyin) -->
    <v-group v-if="isDrawn" :config="{ x: config.x, y: config.y }">
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
        @delete="emit('delete-slot-area', area.id)"
        @rotate="emit('rotate-slot-area', area.id)"
        @edit="emit('edit-slot-area', $event)"
        @update-position="(pos) => emit('update-slot-area-position', { id: area.id, ...pos })"
      />
    </v-group>
  </v-group>
</template>
