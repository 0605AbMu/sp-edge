<script setup lang="ts">
import { computed } from 'vue';

const props = defineProps<{
  type: 'horizontal' | 'vertical';
  offset: number; // Stage offset (x or y)
  scale: number;  // Current zoom scale
  pixelsPerMeter: number;
  viewSize: number; // Container width or height
  leftOffset?: number; // Sidebar width for vertical ruler
  mousePos?: number;
  showIndicator?: boolean;
}>();

const RULER_THICKNESS = 25; // Chizg'ich qalinligi (px)

// Metrlar bo'yicha chiziqlar ro'yxatini hisoblash
const markings = computed(() => {
  const result = [];
  const ppm = props.pixelsPerMeter * props.scale;
  
  // Ekranda ko'rinadigan metrlar oralig'ini aniqlash
  const startM = Math.floor(-props.offset / ppm);
  const endM = Math.ceil((props.viewSize - props.offset) / ppm);
  
  // Masshtabga qarab qadamni aniqlash
  let step = 1;
  if (ppm < 20) step = 2;
  if (ppm < 10) step = 5;
  if (ppm < 5) step = 10;
  if (ppm < 2) step = 50;

  const start = Math.floor(startM / step) * step;
  
  for (let m = start; m <= endM; m += step) {
    const pos = props.offset + m * ppm;
    
    // Agar vertikal chizg'ich bo'lsa va leftOffset bo'lsa, uni boshlanishidan chizamiz
    if (pos >= 0 && pos <= props.viewSize) {
      result.push({
        value: m,
        pos: pos,
        isMajor: m % (step * 5) === 0 || step > 1
      });
    }
  }
  return result;
});

const mouseMeter = computed(() => {
  if (props.mousePos === undefined) return null;
  const ppm = props.pixelsPerMeter * props.scale;
  const effectiveMousePos = props.type === 'horizontal' 
    ? props.mousePos - (props.leftOffset || 0) 
    : props.mousePos;
  const val = (effectiveMousePos - props.offset) / ppm;
  return val.toFixed(2);
});
</script>

<template>
  <div 
    :class="['ruler', type]" 
    :style="{ 
      width: type === 'horizontal' ? (leftOffset ? `calc(100% - ${leftOffset}px)` : '100%') : RULER_THICKNESS + 'px',
      height: type === 'vertical' ? '100%' : RULER_THICKNESS + 'px',
      left: leftOffset ? leftOffset + 'px' : 0
    }"
  >
    <div 
      v-for="mark in markings" 
      :key="mark.value" 
      class="mark"
      :style="type === 'horizontal' ? { left: mark.pos + 'px' } : { top: mark.pos + 'px' }"
    >
      <div :class="['line', { major: mark.isMajor }]"></div>
      <span v-if="mark.isMajor" class="label">{{ mark.value }}m</span>
    </div>

    <!-- Mouse indicator line -->
    <div 
      v-if="showIndicator && mousePos !== undefined" 
      class="mouse-indicator"
      :style="type === 'horizontal' 
        ? { left: (mousePos - (leftOffset || 0)) + 'px' } 
        : { top: mousePos + 'px' }"
    >
      <div class="indicator-label">{{ mouseMeter }}m</div>
      <div class="indicator-line"></div>
    </div>
  </div>
</template>

<style scoped>
.ruler {
  position: absolute;
  background: rgba(255, 255, 255, 0.85);
  backdrop-filter: blur(4px);
  z-index: 10;
  pointer-events: none;
}

.horizontal {
  top: 0;
  right: 0;
  border-left: none;
  border-top: none;
  border-bottom: 1px solid #999;
}

.vertical {
  top: 0;
  bottom: 0;
  border-top: none;
  border-left: none;
  border-right: 1px solid #999;
}

.mark {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
}

.horizontal .mark {
  flex-direction: column;
  height: 100%;
  transform: translateX(-50%);
}

.vertical .mark {
  flex-direction: row;
  width: 100%;
  transform: translateY(-50%);
}

.line {
  background: #999;
}

.horizontal .line {
  width: 1.5px;
  height: 8px;
}

.horizontal .line.major {
  height: 15px;
  background: #333;
}

.vertical .line {
  height: 1.5px;
  width: 8px;
}

.vertical .line.major {
  width: 15px;
  background: #333;
}

.label {
  font-size: 9px;
  color: #333;
  font-weight: bold;
  pointer-events: none;
  user-select: none;
}

.horizontal .label {
  margin-top: 2px;
}

.vertical .label {
  margin-left: 2px;
  writing-mode: vertical-rl;
  transform: rotate(180deg);
}

/* Mouse indicator styles */
.mouse-indicator {
  position: absolute;
  z-index: 20;
  display: flex;
  align-items: center;
  pointer-events: none;
}

.horizontal .mouse-indicator {
  flex-direction: column;
  top: 0;
  height: 200vh; /* Stage bo'ylab chiziq (kattaroq masshtab uchun) */
  transform: translateX(-50%);
  overflow: visible;
}

.vertical .mouse-indicator {
  flex-direction: row;
  left: 0;
  width: 200vw; /* Stage bo'ylab chiziq (kattaroq masshtab uchun) */
  transform: translateY(-50%);
  overflow: visible;
}

.indicator-line {
  background: rgba(255, 68, 68, 0.6); /* Bir oz to'qroq qizil chiziq */
}

.horizontal .indicator-line {
  width: 1px;
  height: 100%;
}

.vertical .indicator-line {
  height: 1px;
  width: 100%;
}

.indicator-label {
  background: rgba(255, 68, 68, 0.8);
  color: white;
  font-size: 10px;
  padding: 1px 4px;
  border-radius: 3px;
  font-weight: bold;
}

.horizontal .indicator-label {
  margin-bottom: 2px;
}

.vertical .indicator-label {
  margin-right: 2px;
  writing-mode: vertical-rl;
  transform: rotate(180deg);
}
</style>
