<template>
  <div class="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 text-gray-900">
    <div class="max-w-7xl mx-auto p-6">
      <h1 class="text-3xl font-bold mb-6 tracking-tight">Weekly Calendar Demo</h1>

      <WeeklyCalendar v-model="schedule" />

      <div class="mt-8">
        <h2 class="font-semibold mb-2">Current model value</h2>
        <pre class="p-4 bg-white rounded-lg border shadow-sm overflow-auto text-sm">{{ schedule }}</pre>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import WeeklyCalendar from './components/WeeklyCalendar.vue'

const schedule = ref({
  mo: [{ bt: 240, et: 779 }],
  tu: [],
  we: [],
  th: [{ bt: 240, et: 779 }, { bt: 1140, et: 1319 }],
  fr: [{ bt: 660, et: 1019 }],
  sa: [{ bt: 0, et: 1439 }],
  su: [],
})
</script>
