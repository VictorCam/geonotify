<script setup>
import { ref, computed, watch } from 'vue'
import { useFetch, useDebounceFn, useGeolocation, useShare, throttledRef, isDefined } from '@vueuse/core'

const DEBOUNCE_TIME = 2000;
const COORD_THROTTLE_TIME = 4000;
const isLengthZero = (str) => str.length === 0

const { share } = useShare()
const { coords, error, resume, pause } = useGeolocation()
pause()

const throttledCoords = throttledRef(coords, COORD_THROTTLE_TIME)

const address1 = ref('')
const address2 = ref('')
const useGPS = ref(false)
const url1 = ref(``)
const url2 = ref(``)
const url3 = ref(``) //calculate the distance

const { data: data1, error: err1, isFetching: isFetching1, execute: exec1 } = useFetch(url1, { immediate: false }).json()
const { data: data2, error: err2, isFetching: isFetching2, execute: exec2 } = useFetch(url2, { immediate: false }).json()
const { data: data3, error: err3, isFetching: isFetching3, execute: exec3 } = useFetch(url3, { immediate: false }).json()

const toggleShareAnim = computed(() => data3.value?.routes[0].duration ? 'jumpout approval-btn' : 'shake-horizontal denial-btn cursor-not-allowed')
const toggleGPS = computed(() => useGPS.value ? 'svg-color bg-green-700 .dark:bg-green checkmark' : 'svg-color bg-red close')
const toggleGPSAnim = computed(() => useGPS.value ? 'jumpout approval-btn' : 'shake-horizontal denial-btn')
const roundedTime = computed(() => (data3.value?.routes[0].duration / 60).toFixed(2))

const searchLocation1 = () => {
  if (isLengthZero(address1.value.trim())) return
  url1.value = `https://nominatim.openstreetmap.org/search?q=${address1.value}&format=json`
  exec1()
}
const searchLocation2 = () => {
  if (isLengthZero(address2.value.trim())) return
  url2.value = `https://nominatim.openstreetmap.org/search?q=${address2.value}&format=json`
  exec2()
}

const resetValues = () => {
  if (useGPS.value == true) {
    data1.value = null
    err1.value = null
    resume()
  }
  else {
    pause()
    address1.value = ''
    data3.value = null
    err3.value = null
  }
}
const handleDataChange = () => {
  if (isDefined(data1) && isDefined(data2)) {
    let lat1 = data1.value[0].lat
    let lon1 = data1.value[0].lon
    let lat2 = data2.value[0].lat
    let lon2 = data2.value[0].lon
    url3.value = `https://router.project-osrm.org/route/v1/driving/${lon1},${lat1};${lon2},${lat2}`
    exec3()
  }
}
const handleCoordsChange = () => {
  if (data2.value?.[0] && useGPS.value == true
    && throttledCoords.value != Infinity && isFetching3.value == false) {
    let lat1 = throttledCoords.value.latitude
    let lon1 = throttledCoords.value.longitude
    let lat2 = data2.value[0].lat
    let lon2 = data2.value[0].lon
    url3.value = `https://router.project-osrm.org/route/v1/driving/${lon1},${lat1};${lon2},${lat2}`
    exec3()
  }
}

watch([useGPS], resetValues)
watch([data1, data2], handleDataChange)
watch([data2, throttledCoords], handleCoordsChange)

let debounceSearch1 = useDebounceFn(searchLocation1, DEBOUNCE_TIME)
let debounceSearch2 = useDebounceFn(searchLocation2, DEBOUNCE_TIME)


const shareTime = () => {
  share({
    text: 'I will approximately arrive in ' + roundedTime.value + ' minutes',
  })
}
</script>

<template>
  <div>
    <NavHeader></NavHeader>
    <div class="center <sm:w80vw bg-black/15 shadow-xl items-start flex flex-col shadow-black/30 p5 rd-3">

      <!-- serach Location -->
      <div class="pos-input">
        <p class="font-bold w25">Location:</p>
        <input :disabled="useGPS" v-model="address1" @input="debounceSearch1" />
      </div>

      <!-- result -->
      <TransitionGroup class="wfit mb3 font-bold" name="list" tag="p">
        <p v-if="isFetching1" class="flex gap2">Loading...
        <div aria-hidden="true" class="svg-color loading"></div>
        </p>
        <p class="text-red" v-else-if="data1?.length == 0">No results</p>
        <p class="text-red" v-else-if="err1">{{ err1 }}</p>
        <p class="c-green-700 .dark:c-green flex gap2" v-else-if="useGPS">Using GPS
        <div aria-hidden="true" class="svg-color bg-green animCheck">
        </div>
        </p>
        <p class="c-green-700 .dark:c-green flex gap2" v-else-if="data1">Found Location
        <div aria-hidden="true" class="svg-color bg-green-900 animCheck">
        </div>
        </p>
      </TransitionGroup>

      <!-- serach Destination -->
      <div class="pos-input">
        <p class="font-bold w25">Destination:</p>
        <input v-model="address2" @input="debounceSearch2" />
      </div>

      <!-- result -->
      <TransitionGroup class="wfit mb3 font-bold" name="list" tag="p">
        <p class="flex gap2" v-if="isFetching2">Loading...
        <div aria-hidden="true" class="svg loading"></div>
        </p>
        <p class="text-red" v-else-if="data2?.length == 0">No results</p>
        <p class="text-red" v-else-if="err2">{{ err2 }}</p>
        <p class="c-green-700 .dark-c-green flex gap2" v-else-if="data2">Found Destination
        <div class="svg-color bg-green-700 .dark:bg-green animCheck">
        </div>
        </p>
      </TransitionGroup>

      <!-- buttons -->
      <div class="flex gap3 items-center flex-wrap">
        <button :class="toggleGPSAnim" @click="useGPS = !useGPS" class="btns">
          Use GPS <div :class="toggleGPS"></div>
        </button>
        <button :disabled="!data3?.routes[0].duration" :class="toggleShareAnim" class="btns" @click="shareTime">
          <p>Share Time</p>
          <div class="svg-color share p0.3"></div>
        </button>

        <!-- final results -->
        <p v-if="isFetching3" class="flex gap2">Calculating...
        <div aria-hidden="true" class="svg loading"></div>
        </p>
        <p v-else-if="data3?.length == 0">No results</p>
        <p v-else-if="err3">{{ err3 }}</p>
        <p v-else-if="data3">You will arrive in
          <span class="font-bold underline"> {{ roundedTime }}</span>
          minutes
        </p>
      </div>
    </div>
  </div>
</template>

<style scoped>
input {
  --at-apply: rd-1 b b-solid p1 .dark:bg-black/20 .dark:text-white bg-white/50 text-black;
}

.approval-btn {
  --at-apply: text-green-700 b-green-700 .dark:b-green .dark:text-green;
}

.denial-btn {
  --at-apply: b-red text-red b-red text-red;
}

.btns {
  --at-apply: flex text-nowrap gap2 items-center rd b-2px b-solid p1;
}

.pos-input {
  --at-apply: flex flex-wrap items-center gap3 mb3;
}
</style>