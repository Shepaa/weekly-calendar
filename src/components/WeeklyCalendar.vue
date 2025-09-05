<template>
  <div class="w-full">
    <div class="mb-4 flex items-center justify-between">
      <div>
        <h2 class="text-xl font-semibold">Weekly Schedule</h2>
        <p class="text-sm text-gray-500">Click or drag to select hours. All Day selects the whole day. Clear resets
          all.</p>
      </div>
      <button
          class="px-3 py-2 text-sm rounded-md bg-white border shadow-sm hover:bg-gray-50 active:bg-gray-100 transition"
          @click="clearEverything"
      >
        Clear All
      </button>
    </div>

    <div class="bg-white rounded-lg border shadow-sm overflow-hidden">
      <!-- Header row: empty top-left cell + 24 hours across -->
      <div class="grid" :style="{ gridTemplateColumns: `8rem repeat(24, minmax(2.5rem, 1fr))` }">
        <div
            class="bg-gray-50 border-b border-r h-12 flex items-center justify-center font-semibold text-sm text-gray-500">
          Day / Hour
        </div>
        <div v-for="hour in 24" :key="`h-${hour}`"
             class="bg-gray-50 border-b h-12 flex items-center justify-center text-xs text-gray-700">
          {{ (hour - 1).toString().padStart(2, '0') }}:00
        </div>
      </div>
      <!-- Body: one row per day -->
      <div class="grid" :style="{ gridTemplateColumns: `8rem repeat(24, minmax(2.5rem, 1fr))` }">
        <template v-for="(dayKey, dayIndex) in dayOrder" :key="dayKey">
          <!-- Day label + All Day button on the left -->
          <div class="flex items-center justify-between px-3 gap-2 border-r bg-gray-50 h-10 select-none"
               @contextmenu.prevent="toggleAllDay(dayIndex)"
               @mousedown.stop.prevent="startDayDrag(dayIndex)"
               @mouseenter="onDayEnter(dayIndex)"
               @mouseup.prevent="endDayDrag"
          >
            <div class="font-semibold capitalize text-gray-800">{{ dayNames[dayKey] }}</div>
            <button
                class="text-xs px-2 py-1 border rounded-md bg-white hover:bg-gray-100 active:bg-gray-200 transition"
                @mousedown.stop.prevent
                @mouseup.stop.prevent
                @click.stop="toggleAllDay(dayIndex)"
            >
              All Day
            </button>
          </div>
          <!-- 24 hour cells horizontally -->
          <div v-for="hour in 24" :key="`d-${dayIndex}-h-${hour}`"
               class="cursor-pointer h-10 border-t border-l last:border-r"
               :class="[
                 isSelected(dayIndex, hour-1) ? 'bg-blue-600 hover:bg-blue-700' : 'bg-gray-100 hover:bg-gray-200',
               ]"
               @mousedown.stop.prevent="startDrag(dayIndex, hour-1)"
               @mouseenter="onEnter(dayIndex, hour-1)"
               @contextmenu.prevent="toggleAllDay(dayIndex)"
               @mouseup.prevent="endDrag"
          ></div>
        </template>
      </div>
    </div>
  </div>
</template>

<script setup>
import {watch, reactive, ref} from 'vue'

const props = defineProps({
  modelValue: {
    type: Object,
    default: () => ({mo: [], tu: [], we: [], th: [], fr: [], sa: [], su: []})
  }
})
const emit = defineEmits(['update:modelValue'])

const dayOrder = ['mo', 'tu', 'we', 'th', 'fr', 'sa', 'su']
const dayNames = {
  mo: 'Mo',
  tu: 'Tu',
  we: 'We',
  th: 'Th',
  fr: 'Fr',
  sa: 'Sa',
  su: 'Su',
}

const selected = reactive(Array.from({length: 7}, () => Array.from({length: 24}, () => false)))

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
    intervals.forEach(({bt, et}) => {
      const startHour = Math.floor(bt / 60)
      const endHour = Math.floor(et / 60)
      for (let h = startHour; h <= endHour && h < 24; h++) {
        selected[dIndex][h] = true
      }
    })
  })
}


applyFromModel(props.modelValue)

watch(() => props.modelValue, (nv) => {
  applyFromModel(nv)
})

function isSelected(d, h) {
  return selected[d][h]
}


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

// Drag selection logic (day-level)
const dayDragging = ref(false)
const dayDragMode = ref('select') // or 'clear'

function isDayFullySelected(d) {
  return selected[d].every(Boolean)
}

function setDay(d, value) {
  for (let h = 0; h < 24; h++) selected[d][h] = value
}

function startDayDrag(d) {
  dayDragging.value = true
  dayDragMode.value = isDayFullySelected(d) ? 'clear' : 'select'
  setDay(d, dayDragMode.value === 'select')
  emitSchedule()
}

function onDayEnter(d) {
  if (!dayDragging.value) return
  setDay(d, dayDragMode.value === 'select')
}

function endDayDrag() {
  if (!dayDragging.value) return
  dayDragging.value = false
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
        arr.push({bt, et})
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
    if (dayDragging.value) endDayDrag()
  })
}
</script>

<style scoped>
/**** no extra styles; tailwind used ****/
</style>
