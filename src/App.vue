<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const seconds = ref(0)
const isRunning = ref(false)
const isBlowing = ref(false)
const isEnded = ref(false)
let timer = null
let pausedSeconds = 0       // 暂停时已累计秒数
let startWallTime = null    // 开始/恢复时的 Date.now()，独立于 setInterval

const formattedTime = computed(() => {
  const hrs = Math.floor(seconds.value / 3600)
  const mins = Math.floor((seconds.value % 3600) / 60)
  const secs = seconds.value % 60
  return `${String(hrs).padStart(2, '0')}:${String(mins).padStart(2, '0')}:${String(secs).padStart(2, '0')}`
})

const growth = computed(() => {
  if (isEnded.value) return 0
  return Math.min(seconds.value / 900, 1)
})

const isSwaying = computed(() => {
  return growth.value > 0 && !isBlowing.value && !isEnded.value
})

// 生成种子分布
const generateSeeds = () => {
  const seeds = []
  const layers = 5
  let seedIndex = 0

  for (let layer = 0; layer < layers; layer++) {
    const baseRadius = 25 + layer * 15
    const seedsInLayer = 8 + layer * 4

    for (let i = 0; i < seedsInLayer; i++) {
      const baseAngle = (i / seedsInLayer) * Math.PI * 2 + (layer * 0.3)
      const angle = baseAngle + (Math.random() - 0.5) * 0.4
      const radius = baseRadius + (Math.random() - 0.5) * 12

      const centerX = 200
      const centerY = 250

      const baseX = centerX + Math.cos(angle) * radius
      const baseY = centerY + Math.sin(angle) * radius * (0.75 + Math.random() * 0.15)

      const stemLength = 16 + Math.random() * 6
      const headX = baseX - Math.cos(angle) * stemLength
      const headY = baseY - Math.sin(angle) * stemLength

      const startDistance = 400 + Math.random() * 200
      const startX = centerX + Math.cos(angle) * startDistance
      const startY = centerY + Math.sin(angle) * startDistance

      const endAngle = angle + (Math.random() - 0.5) * 0.5
      const endDistance = 500 + Math.random() * 300
      const endX = centerX + Math.cos(endAngle) * endDistance
      const endY = centerY + Math.sin(endAngle) * endDistance - Math.random() * 150

      seeds.push({
        id: `${layer}-${i}`,
        baseX, baseY, headX, headY,
        startX, startY, endX, endY,
        angle, layer, seedIndex
      })

      seedIndex++
    }
  }

  return seeds.sort(() => Math.random() - 0.5).map((seed, index) => ({
    ...seed,
    delay: index * 0.05
  }))
}

let allSeeds = generateSeeds()

const visibleSeeds = computed(() => {
  const seedsToShow = Math.floor(growth.value * allSeeds.length)
  return allSeeds.slice(0, seedsToShow)
})

// 根据 startWallTime 和 pausedSeconds 同步 seconds
const syncSeconds = () => {
  if (startWallTime !== null) {
    seconds.value = pausedSeconds + Math.floor((Date.now() - startWallTime) / 1000)
  }
}

const start = () => {
  if (isEnded.value) {
    seconds.value = 0
    pausedSeconds = 0
    isEnded.value = false
    allSeeds = generateSeeds()
  }

  if (!isRunning.value) {
    isRunning.value = true
    isBlowing.value = false
    startWallTime = Date.now()
    // setInterval 仅用于驱动 UI 刷新，精度由 startWallTime 保证
    timer = setInterval(syncSeconds, 500)
  }
}

const pause = () => {
  isRunning.value = false
  if (timer) {
    clearInterval(timer)
    timer = null
  }
  syncSeconds()           // 暂停前先同步一次，确保秒数准确
  pausedSeconds = seconds.value
  startWallTime = null
}

const reset = () => {
  pause()
  isBlowing.value = true
  setTimeout(() => {
    isBlowing.value = false
    isEnded.value = true
  }, 3000)
}

// 处理页面可见性变化
const handleVisibilityChange = () => {
  if (!document.hidden && isRunning.value) {
    // 页面重新可见：直接用 startWallTime 计算精确秒数，无需补偿
    syncSeconds()

    // 重新触发蒲公英摇晃动画
    const dandelion = document.querySelector('.dandelion')
    if (dandelion && dandelion.classList.contains('swaying')) {
      dandelion.classList.remove('swaying')
      void dandelion.offsetWidth
      dandelion.classList.add('swaying')
    }
  }
}

onMounted(() => {
  document.addEventListener('visibilitychange', handleVisibilityChange)
})

onUnmounted(() => {
  if (timer) clearInterval(timer)
  document.removeEventListener('visibilitychange', handleVisibilityChange)
})
</script>

<template>
  <div class="app-container">
    <div class="timer-section">
      <div class="timer-display" :class="{ 'timer-active': isRunning }">{{ formattedTime }}</div>
      <div class="controls">
        <button @click="start" :disabled="isRunning" class="btn btn-start">开始</button>
        <button @click="pause" :disabled="!isRunning" class="btn btn-pause">暂停</button>
        <button @click="reset" class="btn btn-reset">结束</button>
      </div>
    </div>

    <div class="dandelion-section">
      <svg
        class="dandelion"
        :class="{ 'swaying': isSwaying }"
        viewBox="0 0 400 500"
        xmlns="http://www.w3.org/2000/svg"
      >
        <!-- 茎 -->
        <g v-if="growth > 0" :class="{ 'fade-out': isBlowing }">
          <path
            d="M 200 500 Q 185 400, 190 300 Q 195 275, 200 250"
            stroke="#1a1a1a"
            stroke-width="2.5"
            stroke-linecap="round"
            fill="none"
            class="stem"
          />
        </g>

        <!-- 种子 -->
        <g v-for="seed in visibleSeeds" :key="seed.id">
          <g
            :class="{ 'seed-blowing': isBlowing }"
            class="seed-group"
            :style="{
              '--start-x': `${seed.startX - seed.baseX}px`,
              '--start-y': `${seed.startY - seed.baseY}px`,
              '--end-x': `${seed.endX - seed.baseX}px`,
              '--end-y': `${seed.endY - seed.baseY}px`,
              '--delay': `${seed.delay}s`,
              '--blow-delay': `${(4 - seed.layer) * 0.15 + (seed.seedIndex % 8) * 0.02}s`
            }"
          >
            <!-- 伞毛 -->
            <g v-for="i in 12" :key="`hair-${i}`">
              <line
                :x1="seed.baseX"
                :y1="seed.baseY"
                :x2="seed.baseX + Math.cos((i / 12) * Math.PI * 2) * 10"
                :y2="seed.baseY + Math.sin((i / 12) * Math.PI * 2) * 10"
                stroke="#333"
                stroke-width="0.4"
                stroke-linecap="round"
                opacity="0.6"
              />
              <circle
                :cx="seed.baseX + Math.cos((i / 12) * Math.PI * 2) * 10"
                :cy="seed.baseY + Math.sin((i / 12) * Math.PI * 2) * 10"
                r="0.6"
                fill="#333"
                opacity="0.5"
              />
            </g>

            <!-- 种子柄 -->
            <line
              :x1="seed.baseX"
              :y1="seed.baseY"
              :x2="seed.headX"
              :y2="seed.headY"
              stroke="#1a1a1a"
              stroke-width="0.6"
              stroke-linecap="round"
            />

            <!-- 种子本体 -->
            <ellipse
              :cx="seed.headX"
              :cy="seed.headY"
              rx="1.5"
              ry="2.5"
              fill="#1a1a1a"
              :transform="`rotate(${(seed.angle + Math.PI) * 180 / Math.PI} ${seed.headX} ${seed.headY})`"
            />
          </g>
        </g>
      </svg>
    </div>
  </div>
</template>

<style scoped>
.app-container {
  width: 100%;
  height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}

.timer-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 3rem;
  z-index: 10;
}

.timer-display {
  font-size: 5rem;
  font-weight: 300;
  letter-spacing: 0.1em;
  color: var(--text-primary);
  font-variant-numeric: tabular-nums;
  transition: color 0.3s ease;
}

.timer-display.timer-active {
  color: #999;
}

.controls {
  display: flex;
  gap: 1.5rem;
}

.btn {
  padding: 0.8rem 2rem;
  font-size: 1rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  font-weight: 500;
  letter-spacing: 0.05em;
}

.btn:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.btn-start, .btn-pause {
  background: var(--text-primary);
  color: var(--bg);
}

.btn-start:not(:disabled):hover,
.btn-pause:not(:disabled):hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}

.btn-reset {
  background: transparent;
  color: var(--text-primary);
  border: 2px solid var(--text-primary);
}

.btn-reset:hover {
  background: var(--text-primary);
  color: var(--bg);
  transform: translateY(-2px);
}

.dandelion-section {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  width: 100%;
  height: 500px;
  overflow: hidden;
  pointer-events: none;
}

.dandelion {
  width: 100%;
  max-width: 600px;
  height: 500px;
  transform-origin: 200px 500px;
}

.dandelion.swaying {
  animation: sway 4s ease-in-out infinite;
}

.stem {
  animation: growStem 1.2s cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

.seed-group {
  animation: gatherSeed 1.8s cubic-bezier(0.25, 0.46, 0.45, 0.94) forwards;
  animation-delay: var(--delay);
  opacity: 0;
  transform-origin: center;
}

.seed-blowing {
  animation: blowSeed 2.5s cubic-bezier(0.4, 0, 0.6, 1) forwards !important;
  animation-delay: var(--blow-delay) !important;
}

.fade-out {
  animation: fadeOut 2s ease-out forwards;
}

@keyframes growStem {
  from {
    stroke-dasharray: 350;
    stroke-dashoffset: 350;
    opacity: 0;
  }
  to {
    stroke-dasharray: 350;
    stroke-dashoffset: 0;
    opacity: 1;
  }
}

@keyframes gatherSeed {
  0% {
    transform: translate(var(--start-x), var(--start-y)) scale(0.5) rotate(-180deg);
    opacity: 0;
  }
  30% {
    transform: translate(calc(var(--start-x) * 0.6), calc(var(--start-y) * 0.5)) scale(0.6) rotate(-90deg);
    opacity: 0.4;
  }
  60% {
    transform: translate(calc(var(--start-x) * 0.3), calc(var(--start-y) * 0.2)) scale(0.8) rotate(-30deg);
    opacity: 0.7;
  }
  100% {
    transform: translate(0, 0) scale(1) rotate(0deg);
    opacity: 1;
  }
}

@keyframes blowSeed {
  0% {
    transform: translate(0, 0) scale(1) rotate(0deg);
    opacity: 1;
  }
  15% {
    transform: translate(calc(var(--end-x) * 0.08), calc(var(--end-y) * 0.08)) scale(0.98) rotate(30deg);
    opacity: 0.95;
  }
  100% {
    transform: translate(var(--end-x), var(--end-y)) scale(0.3) rotate(720deg);
    opacity: 0;
  }
}

@keyframes fadeOut {
  0% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
  100% {
    opacity: 0;
  }
}

@keyframes sway {
  0%, 100% {
    transform: rotate(0deg);
  }
  25% {
    transform: rotate(2deg);
  }
  50% {
    transform: rotate(0deg);
  }
  75% {
    transform: rotate(-2deg);
  }
}
</style>
