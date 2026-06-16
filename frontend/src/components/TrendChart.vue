<template>
  <div class="flex flex-col gap-2">
    <div class="flex items-center justify-between flex-wrap gap-2">
      <div class="flex items-center gap-3 flex-wrap">
        <span class="text-xs text-gray-500">指标:</span>
        <label v-for="reg in availableRegisters" :key="reg.name" class="flex items-center gap-1 text-xs text-gray-300 cursor-pointer hover:text-orange-400">
          <input type="checkbox" :checked="store.selectedMetricNames.has(reg.name)" @change="store.toggleMetric(reg.name)" class="accent-orange-500" />
          <span>{{ reg.name }}</span>
        </label>
      </div>
      <div class="flex items-center gap-1">
        <span class="text-xs text-gray-500 mr-1">窗口:</span>
        <button v-for="w in timeWindows" :key="w.value" @click="store.setTimeWindow(w.value)"
          class="px-2 py-0.5 rounded text-xs"
          :class="store.timeWindowMinutes === w.value ? 'bg-orange-600 text-white' : 'bg-gray-800 text-gray-400 hover:bg-gray-700'">
          {{ w.label }}
        </button>
      </div>
    </div>
    <v-chart v-if="chartOption" :option="chartOption" class="h-56" autoresize />
    <div v-else class="h-56 flex items-center justify-center text-gray-600 text-sm">
      {{ emptyMessage }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { LineChart } from 'echarts/charts'
import { GridComponent, TooltipComponent, LegendComponent } from 'echarts/components'
import VChart from 'vue-echarts'
import { useModbusStore } from '../store/modbus'
import type { EChartsOption } from 'echarts'
import type { ModbusRegister } from '../types'

use([CanvasRenderer, LineChart, GridComponent, TooltipComponent, LegendComponent])

const store = useModbusStore()

const timeWindows = [
  { label: '5分', value: 5 },
  { label: '15分', value: 15 },
  { label: '30分', value: 30 },
  { label: '1小时', value: 60 },
]

const availableRegisters = computed<ModbusRegister[]>(() => {
  return store.selectedDevice?.registers ?? []
})

const emptyMessage = computed(() => {
  if (!store.selectedDevice) return '选择设备后显示趋势'
  if (store.selectedMetricNames.size === 0) return '请至少选择一个指标'
  if (!store.isPolling) return '开始采集后显示趋势'
  return '暂无数据'
})

const chartOption = computed<EChartsOption | null>(() => {
  const dev = store.selectedDevice
  if (!dev) return null
  if (store.selectedMetricNames.size === 0) return null

  const colors = ['#f97316', '#22d3ee', '#a78bfa', '#34d399']
  const series: any[] = []
  const now = Date.now()
  const cutoff = now - store.timeWindowMs

  dev.registers.forEach((reg, i) => {
    if (!store.selectedMetricNames.has(reg.name)) return
    const key = `${dev.id}_${reg.address}`
    const hd = store.historyData[key]
    if (!hd || !hd.values.length) return

    const filteredTime: number[] = []
    const filteredValues: number[] = []
    for (let j = 0; j < hd.time.length; j++) {
      if (hd.time[j] >= cutoff) {
        filteredTime.push(hd.time[j])
        filteredValues.push(hd.values[j])
      }
    }
    if (filteredTime.length === 0) return

    series.push({
      name: reg.name,
      type: 'line',
      showSymbol: false,
      smooth: true,
      lineStyle: { color: colors[i % colors.length], width: 2 },
      data: filteredTime.map((t, j) => [t, filteredValues[j]])
    })
  })

  if (!series.length) return null

  return {
    tooltip: { trigger: 'axis' },
    legend: { textStyle: { color: '#999' }, top: 0 },
    grid: { left: 50, right: 20, top: 30, bottom: 25 },
    xAxis: {
      type: 'value',
      axisLabel: { color: '#666', formatter: (v: number) => new Date(v).toLocaleTimeString() },
      splitLine: { lineStyle: { color: '#1f2937' } }
    },
    yAxis: { type: 'value', axisLabel: { color: '#666' }, splitLine: { lineStyle: { color: '#1f2937' } } },
    series
  }
})
</script>
