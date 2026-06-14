<template>
  <div class="flex flex-col gap-2">
    <div v-if="store.manualPickMode" class="bg-yellow-500/20 border border-yellow-500 rounded-lg px-4 py-2 text-sm text-yellow-300 flex items-center justify-between">
      <span>
        🎯 手动补拾取模式：点击波形添加
        <span :class="store.manualPickMode === 'P' ? 'text-red-400 font-bold' : 'text-blue-400 font-bold'">
          {{ store.manualPickMode }} 波
        </span>
        到时点
      </span>
      <button @click="store.setManualPickMode(null)" class="text-xs bg-yellow-600 hover:bg-yellow-500 px-3 py-1 rounded text-white">
        退出模式
      </button>
    </div>
    <div v-for="(comp, idx) in components" :key="comp.key" class="bg-gray-900 rounded-xl p-3">
      <h3 class="text-xs text-gray-400 mb-1">{{ comp.label }}</h3>
      <v-chart
        :option="getChartOption(comp.key, comp.color)"
        class="h-40"
        autoresize
        :class="{ 'cursor-crosshair': !!store.manualPickMode }"
        @click="(e: any) => onChartClick(e, comp.key)"
        ref="chartRefs"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { LineChart } from 'echarts/charts'
import { GridComponent, TooltipComponent, MarkLineComponent } from 'echarts/components'
import VChart from 'vue-echarts'
import { useSeismicStore } from '../store/seismic'
import type { EChartsOption } from 'echarts'

use([CanvasRenderer, LineChart, GridComponent, TooltipComponent, MarkLineComponent])

const store = useSeismicStore()

const components = [
  { key: 'bhz' as const, label: 'BHZ (垂直分量)', color: '#22d3ee' },
  { key: 'bhn' as const, label: 'BHN (南北分量)', color: '#a78bfa' },
  { key: 'bhe' as const, label: 'BHE (东西分量)', color: '#34d399' },
]

function onChartClick(params: any, _key: 'bhz' | 'bhn' | 'bhe') {
  if (!store.manualPickMode || !params || typeof params.value?.[0] !== 'number') return
  const clickTime = params.value[0] as number
  store.addManualPick(store.manualPickMode, clickTime)
}

function getChartOption(key: 'bhz' | 'bhn' | 'bhe', color: string): EChartsOption {
  const wf = store.waveform!
  const step = 5
  const time = wf.time.filter((_, i) => i % step === 0)
  const data = wf[key].filter((_, i) => i % step === 0)

  const markLines = store.picks.map(p => {
    const isManual = p.method === 'Manual'
    return {
      xAxis: p.time,
      label: {
        formatter: isManual ? `${p.type}*` : p.type,
        color: p.type === 'P' ? '#ef4444' : '#3b82f6',
        fontSize: 14,
        fontWeight: 'bold' as const,
        backgroundColor: isManual ? 'rgba(239, 68, 68, 0.15)' : 'transparent',
        padding: isManual ? [2, 4] : 0,
        borderRadius: isManual ? 3 : 0,
      },
      lineStyle: {
        color: p.type === 'P' ? '#ef4444' : '#3b82f6',
        width: isManual ? 3 : 2,
        type: isManual ? 'solid' as const : 'dashed' as const,
        shadowBlur: isManual ? 8 : 0,
        shadowColor: p.type === 'P' ? 'rgba(239, 68, 68, 0.5)' : 'rgba(59, 130, 246, 0.5)',
      }
    }
  })

  return {
    tooltip: {
      trigger: 'axis',
      formatter: (params: any) => {
        const t = params[0].value[0].toFixed(2)
        const amp = params[0].value[1].toFixed(4)
        const modeHint = store.manualPickMode
          ? `<br/><span style="color:${store.manualPickMode === 'P' ? '#ef4444' : '#3b82f6'};font-weight:bold">点击添加${store.manualPickMode}波到时</span>`
          : ''
        return `t=${t}s<br/>amp=${amp}${modeHint}`
      }
    },
    grid: { left: 50, right: 20, top: 10, bottom: 25 },
    xAxis: { type: 'value', min: 0, max: time[time.length - 1], axisLabel: { color: '#666', formatter: '{value}s' } },
    yAxis: { type: 'value', axisLabel: { color: '#666' }, splitLine: { lineStyle: { color: '#1f2937' } } },
    series: [{
      type: 'line',
      showSymbol: false,
      lineStyle: { color, width: 1 },
      areaStyle: { color: { type: 'linear', x: 0, y: 0, x2: 0, y2: 1, colorStops: [{ offset: 0, color: color + '33' }, { offset: 1, color: 'transparent' }] } },
      data: time.map((t, i) => [t, data[i]]),
      markLine: { symbol: 'none', data: markLines }
    }]
  }
}
</script>

<style scoped>
.cursor-crosshair {
  cursor: crosshair;
}
</style>
