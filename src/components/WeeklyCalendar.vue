<template>
  <div class="w-full">
    <div class="mb-4 flex items-center justify-between">
      <div>
        <h2 class="text-xl font-semibold">Weekly Schedule</h2>
        <p class="text-sm text-gray-500">Click or drag to select hours. All Day selects the whole day. Clear resets all.</p>
      </div>
      <button
        class="px-3 py-2 text-sm rounded-md bg-white border shadow-sm hover:bg-gray-50 active:bg-gray-100 transition"
        @click="clearEverything"
      >
        Clear All
      </button>
    </div>

    <div class="grid grid-cols-7 gap-3">
      <div v-for="(dayKey, dayIndex) in dayOrder" :key="dayKey" class="flex flex-col bg-white rounded-lg border shadow-sm">
        <div class="flex items-center justify-between px-3 py-2 border-b bg-gray-50 rounded-t-lg">
          <div class="font-semibold capitalize text-gray-800">{{ dayNames[dayKey] }}</div>
          <button
            class="text-xs px-2 py-1 border rounded-md bg-white hover:bg-gray-100 active:bg-gray-200 transition"
            @click="toggleAllDay(dayIndex)"
          >
            All Day
          </button>
        </div>
        <div
          class="grid grid-rows-24 gap-px bg-gray-200 rounded-b-lg overflow-hidden select-none"
          @mousedown.prevent="startDrag(dayIndex, null)"
          @mouseup.prevent="endDrag"
          @mouseleave="endDrag"
        >
          <div
            v-for="hour in 24"
            :key="hour"
            class="cursor-pointer border-t last:border-b border-white"
            :class="[
              isSelected(dayIndex, hour-1) ? 'bg-blue-600 hover:bg-blue-700' : 'bg-gray-100 hover:bg-gray-200',
            ]"
            @mousedown.stop.prevent="startDrag(dayIndex, hour-1)"
            @mouseenter="onEnter(dayIndex, hour-1)"
            @click.prevent="toggleSlot(dayIndex, hour-1)"
          >
            <div class="h-10"></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, watch, reactive, ref } from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({ mo: [], tu: [], we: [], th: [], fr: [], sa: [], su: [] })
  }
})
const emit = defineEmits(['update:modelValue'])

const dayOrder = ['mo','tu','we','th','fr','sa','su']
const dayNames = {
  mo: 'Mo',
  tu: 'Tu',
  we: 'We',
  th: 'Th',
  fr: 'Fr',
  sa: 'Sa',
  su: 'Su',
}

// Internal selected state: 7 days x 24 hours
const selected = reactive(Array.from({ length: 7 }, () => Array.from({ length: 24 }, () => false)))

function clearAll() {
  for (let d = 0; d < 7; d++) {
    for (let h = 0; h < 24; h++) selected[d][h] = false
  }
}

function clearEverything() {
  clearAll()
  emitSchedule()
}

function applyFromModel(model) {
  clearAll()
  dayOrder.forEach((key, dIndex) => {
    const intervals = model?.[key] || []
    intervals.forEach(({ bt, et }) => {
      const startHour = Math.floor(bt / 60)
      const endHour = Math.floor(et / 60)
      for (let h = startHour; h <= endHour && h < 24; h++) {
        selected[dIndex][h] = true
      }
    })
  })
}

// initialize from props
applyFromModel(props.modelValue)

watch(() => props.modelValue, (nv) => {
  applyFromModel(nv)
})

function isSelected(d, h) { return selected[d][h] }

function toggleSlot(d, h) {
  selected[d][h] = !selected[d][h]
  emitSchedule()
}

// Drag selection logic
const dragging = ref(false)
const dragMode = ref('select') // or 'clear'

function startDrag(d, h) {
  dragging.value = true
  if (h === null) return
  dragMode.value = selected[d][h] ? 'clear' : 'select'
  selected[d][h] = (dragMode.value === 'select')
  emitSchedule()
}
function onEnter(d, h) {
  if (!dragging.value) return
  selected[d][h] = (dragMode.value === 'select')
}
function endDrag() {
  if (!dragging.value) return
  dragging.value = false
  emitSchedule()
}

function toggleAllDay(d) {
  const allSelected = selected[d].every(v => v)
  for (let h = 0; h < 24; h++) selected[d][h] = !allSelected
  emitSchedule()
}

function toIntervals() {
  const result = {}
  dayOrder.forEach((key, d) => {
    const arr = []
    let start = null
    for (let h = 0; h < 24; h++) {
      const val = selected[d][h]
      if (val && start === null) start = h
      if ((!val || h === 23) && start !== null) {
        const endHour = val && h === 23 ? h : h - 1
        const bt = start * 60
        const et = endHour * 60 + 59 // inclusive to last minute of hour
        arr.push({ bt, et })
        start = null
      }
    }
    result[key] = arr
  })
  return result
}

function emitSchedule() {
  const value = toIntervals()
  emit('update:modelValue', value)
}

// Global mouseup to end drag if released outside
if (typeof window !== 'undefined') {
  window.addEventListener('mouseup', () => {
    if (dragging.value) endDrag()
  })
}
</script>

<style scoped>
/**** no extra styles; tailwind used ****/
</style>
