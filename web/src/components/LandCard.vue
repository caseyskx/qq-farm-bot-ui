<script setup lang="ts">
import { computed, ref } from 'vue'

const props = defineProps<{
  land: any
}>()

const land = computed(() => props.land)
const showTooltip = ref(false)

function getLandStatusClass(land: any) {
  const status = land.status
  const level = Number(land.level) || 0

  if (status === 'locked')
    return 'bg-gray-100 dark:bg-gray-800 opacity-60 border-dashed'

  let baseClass = 'bg-white dark:bg-gray-800 border-gray-200 dark:border-gray-700'

  // 土地等级样式
  switch (level) {
    case 1: // 黄土地
      baseClass = 'bg-yellow-50/80 dark:bg-yellow-900/10 border-yellow-200 dark:border-yellow-800'
      break
    case 2: // 红土地
      baseClass = 'bg-red-50/80 dark:bg-red-900/10 border-red-200 dark:border-red-800'
      break
    case 3: // 黑土地
      baseClass = 'bg-slate-100 dark:bg-slate-800 border-slate-300 dark:border-slate-600'
      break
    case 4: // 金土地
      baseClass = 'bg-amber-100/80 dark:bg-amber-900/20 border-amber-300 dark:border-amber-600'
      break
  }

  // 状态叠加
  if (status === 'dead')
    return 'bg-gray-200 dark:bg-gray-700 border-gray-300 dark:border-gray-600 grayscale'

  if (status === 'harvestable')
    return `${baseClass} ring-2 ring-yellow-500 ring-offset-1 dark:ring-offset-gray-900`

  if (status === 'stealable')
    return `${baseClass} ring-2 ring-purple-500 ring-offset-1 dark:ring-offset-gray-900`

  if (status === 'growing')
    return baseClass

  return baseClass
}

// 格式化倒计时（精确到秒）
function fmtRemainSec(sec: number) {
  const n = Math.max(0, Math.floor(sec))
  if (n <= 0) return ''
  const h = Math.floor(n / 3600)
  const m = Math.floor((n % 3600) / 60)
  const s = n % 60
  if (h > 0) return `${h}小时${m}分${s}秒`
  if (m > 0) return `${m}分${s}秒`
  return `${s}秒`
}

function getSafeImageUrl(url: string) {
  if (!url)
    return ''
  if (url.startsWith('http://'))
    return url.replace('http://', 'https://')
  return url
}

function getLandTypeName(level: number) {
  const typeMap: Record<number, string> = {
    0: '普通',
    1: '黄土地',
    2: '红土地',
    3: '黑土地',
    4: '金土地',
  }
  return typeMap[Number(level) || 0] || ''
}

// 阶段链数据
const phases = computed(() => {
  const p = land.value?.growPhaseNames
  return Array.isArray(p) ? p : []
})
const curIdx = computed(() => {
  const idx = Number(land.value?.currentPhaseIdx)
  return Number.isFinite(idx) ? idx : -1
})
const hasPhaseChain = computed(() => phases.value.length > 0 && curIdx.value >= 0)
// 紧凑标签：阶段名 (当前/总数)
const phaseLabel = computed(() => {
  if (!hasPhaseChain.value) return ''
  return `${phases.value[curIdx.value]} (${curIdx.value + 1}/${phases.value.length})`
})
</script>

<template>
  <div
    class="relative min-h-[140px] flex flex-col items-center border rounded-lg p-2 transition dark:border-gray-700 hover:shadow-md"
    :class="getLandStatusClass(land)"
  >
    <div class="absolute left-1 top-1 text-[10px] text-gray-400 font-mono">
      #{{ land.id }}
    </div>

    <div class="mb-1 mt-4 h-10 w-10 flex items-center justify-center">
      <img
        v-if="land.seedImage"
        :src="getSafeImageUrl(land.seedImage)"
        class="max-h-full max-w-full object-contain"
        loading="lazy"
        referrerpolicy="no-referrer"
      >
      <div v-else class="i-carbon-sprout text-xl text-gray-300" />
    </div>

    <div class="w-full truncate px-1 text-center text-xs font-bold" :title="land.plantName">
      {{ land.plantName || '-' }}
    </div>

    <!-- 阶段信息 -->
    <div class="relative mb-0.5 mt-0.5 w-full text-center text-[10px] text-gray-500">
      <!-- 有阶段链：显示紧凑标签 + 悬浮气泡 -->
      <template v-if="hasPhaseChain && land.status === 'growing'">
        <span
          class="cursor-pointer text-emerald-500 font-semibold dark:text-emerald-400"
          @mouseenter="showTooltip = true"
          @mouseleave="showTooltip = false"
          @click="showTooltip = !showTooltip"
        >
          {{ phaseLabel }}
        </span>
        <!-- 悬浮气泡 -->
        <div
          v-show="showTooltip"
          class="absolute bottom-full left-1/2 z-50 mb-1.5 -translate-x-1/2 border border-white/15 rounded-md bg-gray-900/95 px-2.5 py-1.5 text-[10px] text-gray-300 shadow-lg whitespace-nowrap"
        >
          <template v-for="(phase, i) in phases" :key="i">
            <span v-if="i > 0" class="mx-0.5 opacity-30">→</span>
            <span
              :class="{
                'text-emerald-400 font-bold opacity-100': i === curIdx,
                'line-through opacity-35': i < curIdx,
                'opacity-50': i > curIdx,
              }"
            >{{ phase }}</span>
          </template>
          <!-- 下阶段倒计时（气泡内） -->
          <div v-if="land.nextPhaseInSec > 0 && land.nextPhaseName" class="mt-1 text-[10px] text-gray-400">
            {{ fmtRemainSec(land.nextPhaseInSec) }}后{{ land.nextPhaseName }}
          </div>
          <!-- 三角箭头 -->
          <div class="absolute left-1/2 top-full -translate-x-1/2 border-5 border-transparent border-t-gray-900/95" />
        </div>
      </template>
      <!-- 已成熟 -->
      <span v-else-if="land.status === 'harvestable'" class="text-yellow-500 font-semibold">
        已成熟
      </span>
      <!-- 枯死 -->
      <span v-else-if="land.status === 'dead'" class="text-red-400 opacity-70">
        枯死
      </span>
      <!-- 无阶段链回退 -->
      <span v-else>
        {{ land.phaseName || (land.status === 'locked' ? '未解锁' : '未开垦') }}
      </span>
    </div>

    <!-- 成熟倒计时（始终显示） -->
    <div v-if="land.matureInSec > 0" class="mb-0.5 w-full text-center text-[10px] text-orange-500 opacity-80">
      {{ fmtRemainSec(land.matureInSec) }}后成熟
    </div>

    <div class="mb-1 text-[10px] text-gray-400">
      {{ getLandTypeName(land.level) }}
    </div>

    <!-- Status Badges -->
    <div class="mt-auto flex origin-bottom scale-90 gap-0.5 text-[10px]">
      <span v-if="land.needWater" class="rounded bg-blue-100 px-0.5 text-blue-700 dark:bg-blue-900/30 dark:text-blue-400">水</span>
      <span v-if="land.needWeed" class="rounded bg-green-100 px-0.5 text-green-700 dark:bg-green-900/30 dark:text-green-400">草</span>
      <span v-if="land.needBug" class="rounded bg-red-100 px-0.5 text-red-700 dark:bg-red-900/30 dark:text-red-400">虫</span>
      <!-- For friends view -->
      <span v-if="land.status === 'harvestable'" class="rounded bg-orange-100 px-0.5 text-orange-700 dark:bg-orange-900/30 dark:text-orange-400">可偷</span>
    </div>
  </div>
</template>
