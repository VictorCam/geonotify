<script setup>
import { ref, computed } from "vue";
import { useDark, useToggle } from '@vueuse/core'
const isDark = useDark();
const toggleDark = useToggle(isDark)
let currIconTheme = computed(() => isDark.value ? 'moon' : 'sun');
</script>

<template>
  <header class="shadow-xl shadow-black/30 bg-black/20 relative z99">
    <div class="sm:mx3 flex items-center gap2 p3 h8 ">
      <h1><span class="bg-clip-text text-transparent bg-gradient-to-r from-blue-400 to-green-500">Geo</span>notify</h1>
      <div class="svg earth p2 mr-auto spin"></div>
      <button @click="toggleDark()" class="p1" :aria-pressed="isDark">
        <div class="svg-color p1 transition duration-300 ease-in" :class="currIconTheme"></div>
      </button>
    </div>
  </header>
</template>