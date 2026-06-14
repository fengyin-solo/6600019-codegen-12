<template>
  <div class="flex h-screen">
    <div class="w-72 bg-gray-900 p-4 flex flex-col gap-3 border-r border-gray-800 overflow-y-auto">
      <h1 class="text-lg font-bold text-cyan-400">地震波形 P/S 波分析</h1>

      <div>
        <label class="block bg-cyan-500 text-black text-center py-2 rounded cursor-pointer hover:bg-cyan-400 text-sm font-medium">
          上传 SAC/miniSEED
          <input type="file" @change="onUpload" class="hidden" />
        </label>
      </div>
      <button @click="store.loadMockData()" class="bg-gray-800 py-2 rounded text-sm hover:bg-gray-700">
        加载模拟数据
      </button>

      <!-- STA/LTA Parameters -->
      <div class="bg-gray-800 rounded-xl p-3 space-y-2">
        <h3 class="text-cyan-300 font-bold text-sm">STA/LTA 参数</h3>
        <div>
          <label class="text-gray-400 text-xs">STA 窗口: {{ store.staWindow.toFixed(1) }}s</label>
          <input type="range" v-model.number="store.staWindow" min="0.5" max="5" step="0.1" class="w-full" />
        </div>
        <div>
          <label class="text-gray-400 text-xs">LTA 窗口: {{ store.ltaWindow.toFixed(1) }}s</label>
          <input type="range" v-model.number="store.ltaWindow" min="5" max="30" step="0.5" class="w-full" />
        </div>
        <div>
          <label class="text-gray-400 text-xs">触发阈值: {{ store.threshold.toFixed(1) }}</label>
          <input type="range" v-model.number="store.threshold" min="1" max="10" step="0.5" class="w-full" />
        </div>
        <button @click="runPick" class="w-full bg-cyan-600 py-2 rounded text-sm hover:bg-cyan-500">
          运行自动拾取
        </button>
      </div>

      <!-- Manual Pick Controls -->
      <div class="bg-gray-800 rounded-xl p-3 space-y-2">
        <h3 class="text-cyan-300 font-bold text-sm">手动补拾取</h3>
        <p class="text-gray-500 text-xs">自动结果不理想时，可直接点击波形手动补录</p>
        <div class="flex gap-2">
          <button
            @click="store.setManualPickMode(store.manualPickMode === 'P' ? null : 'P')"
            class="flex-1 py-2 rounded text-sm font-medium transition-colors"
            :class="store.manualPickMode === 'P'
              ? 'bg-red-500 text-white ring-2 ring-red-300'
              : 'bg-gray-700 text-red-400 hover:bg-gray-600'"
          >
            📍 标记 P 波
          </button>
          <button
            @click="store.setManualPickMode(store.manualPickMode === 'S' ? null : 'S')"
            class="flex-1 py-2 rounded text-sm font-medium transition-colors"
            :class="store.manualPickMode === 'S'
              ? 'bg-blue-500 text-white ring-2 ring-blue-300'
              : 'bg-gray-700 text-blue-400 hover:bg-gray-600'"
          >
            📍 标记 S 波
          </button>
        </div>
        <button
          v-if="store.picks.length > 0"
          @click="store.clearPicks()"
          class="w-full bg-gray-700 py-1.5 rounded text-xs text-gray-400 hover:bg-red-900 hover:text-red-300 transition-colors"
        >
          清空所有拾取
        </button>
      </div>

      <!-- Picks -->
      <div class="bg-gray-800 rounded-xl p-3">
        <div class="flex justify-between items-center mb-2">
          <h3 class="text-cyan-300 font-bold text-sm">震相拾取结果</h3>
          <span v-if="store.picks.length" class="text-xs text-gray-500">{{ store.picks.length }} 个</span>
        </div>
        <div v-for="p in store.picks" :key="p.id" class="bg-gray-700 rounded p-2 mb-1 text-sm">
          <div class="flex justify-between items-center">
            <div class="flex items-center gap-2">
              <span
                :class="p.type === 'P' ? 'text-red-400' : 'text-blue-400'"
                class="font-bold"
              >
                {{ p.type }} 波
                <span v-if="p.method === 'Manual'" class="text-xs text-yellow-400">手动</span>
                <span v-else class="text-xs text-gray-500">自动</span>
              </span>
            </div>
            <button
              @click="store.removePick(p.id)"
              class="text-gray-500 hover:text-red-400 text-xs px-1"
              title="删除"
            >
              ✕
            </button>
          </div>
          <div class="flex justify-between items-center mt-1">
            <span class="text-gray-300 font-mono">{{ p.time.toFixed(2) }}s</span>
            <span class="text-gray-400 text-xs">置信 {{ (p.confidence * 100).toFixed(0) }}%</span>
          </div>
        </div>
        <div v-if="!store.picks.length" class="text-gray-600 text-xs">加载数据后运行拾取或手动标记</div>
      </div>

      <!-- Stations -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">台站分布</h3>
        <div v-for="s in store.stations" :key="s.id"
          @click="store.selectedStation = s"
          class="bg-gray-700 rounded p-2 mb-1 text-sm cursor-pointer hover:bg-gray-600"
          :class="store.selectedStation?.id === s.id ? 'ring-1 ring-cyan-500' : ''">
          {{ s.name }} <span class="text-gray-400 text-xs">({{ s.latitude.toFixed(1) }}, {{ s.longitude.toFixed(1) }})</span>
        </div>
      </div>

      <!-- Events -->
      <div class="bg-gray-800 rounded-xl p-3">
        <h3 class="text-cyan-300 font-bold text-sm mb-2">地震事件目录</h3>
        <div v-for="e in store.events" :key="e.id" class="bg-gray-700 rounded p-2 mb-1 text-xs">
          M{{ e.magnitude }} {{ e.location }}
          <div class="text-gray-500">深度 {{ e.depth }}km | {{ e.originTime.slice(0, 16) }}</div>
        </div>
      </div>
    </div>

    <!-- Main: Waveform Charts -->
    <div class="flex-1 flex flex-col gap-2 p-4 overflow-y-auto">
      <WaveformChart v-if="store.waveform" />
      <div v-else class="flex-1 flex items-center justify-center text-gray-600">
        请上传数据或加载模拟波形
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useSeismicStore } from './store/seismic'
import WaveformChart from './components/WaveformChart.vue'

const store = useSeismicStore()

function onUpload(e: Event) {
  const file = (e.target as HTMLInputElement).files?.[0]
  if (file) store.uploadAndAnalyze(file)
}

function runPick() {
  store.picks = store.staLtaPicking()
}
</script>
