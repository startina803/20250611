<template>
  <v-container>
    <v-row>
      <v-col cols="12">
        <h1>目前事項</h1>
        <h2>{{ list.currentItem }}</h2>
        <h2>{{ list.timeleft }}</h2>
        <h2>{{ timeLeftText }}</h2>
        <DigitNumber v-for="(data, i) in timeLeftText" :key="i" color="white" :data="data" />
      </v-col>
      <v-col cols="12">
        <div class="date-container">
          <h3>日期: {{ currentDate }}</h3> <!-- 日期顯示 -->
        </div>
        <!--
          開始按鈕停用條件:
          1. 倒數中
          2. 目前沒有事項或沒有未完成事項
        -->
        <v-btn
          :disabled="
            status === STATUS.COUNTING || (list.currentItem.length === 0 && list.items.length === 0)
          "
          icon="mdi-play"
          @click="startTimer"
        />
        <!-- 只有倒數中才能按暫停 -->
        <v-btn :disabled="status !== STATUS.COUNTING" icon="mdi-pause" @click="pause" />
        <!-- 目前有事項才能跳過 -->
        <v-btn :disabled="list.currentItem.length === 0" icon="mdi-skip-next" @click="finish" />
      </v-col>
    </v-row>
  </v-container>
</template>

<script setup>
import { useWebNotification } from '@vueuse/core'
import { computed, ref } from 'vue'
import DigitNumber from '@/components/DigitNumber.vue'
import { useListStore } from '@/stores/list'
import { useSettingsStore } from '@/stores/settings'

const list = useListStore()
const settings = useSettingsStore()

// 倒數狀態
const STATUS = {
  STOP: 0,
  COUNTING: 1,
  PAUSE: 2,
}
const status = ref(STATUS.STOP)

// 計時器
let timer = 0
// 開始計時器
// 暫停繼續 + 停止開始
const startTimer = () => {
  // 如果是停止開始，更新目前事項
  if (status.value === STATUS.STOP && list.items.length > 0) {
    list.setCurrentItem()
  }

  // 狀態變成倒數中
  status.value = STATUS.COUNTING

  // 開始倒數
  timer = setInterval(() => {
    list.countdown()

    if (list.timeleft < 0) {
      finish(timer)
    }
  }, 1000)
}

const pause = () => {
  clearInterval(timer)
  status.value = STATUS.PAUSE
}

const finish = () => {
  clearInterval(timer)

  status.value = STATUS.STOP

  const audio = new Audio()
  audio.src = settings.selectedAlarm.file
  audio.play()

  const { show, isSupported } = useWebNotification({
    title: '事項完成',
    body: list.currentItem,
    icon: new URL('@/assets/tomato.png', import.meta.url).href,
  })
  if (isSupported.value) {
    show()
  }

  list.setFinishItem()

  if (list.items.length > 0) {
    startTimer()
  }
}

const timeLeftText = computed(() => {
  const m = Math.floor(list.timeleft / 60)
    .toString()
    .padStart(2, '0')
  const s = (list.timeleft % 60).toString().padStart(2, '0')
  return m + ':' + s
})

// 添加當前日期
const currentDate = computed(() => {
  const now = new Date()
  return now.toLocaleString('en-US', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    hour12: true,
    timeZoneName: 'short',
  }).replace(' at', ',') // 調整格式，去掉 'at'，符合 "Friday, July 11, 2025, 04:15 PM CST"
})
</script>

<style scoped>
.v-btn {
  margin: 0 15px; /* 按鈕間距 */
  border: 2px solid #dc5757;
}

.date-container {
  text-align: right; /* 將日期靠右對齊 */
  margin-bottom: 10px; /* 與按鈕之間的間距 */
  position: absolute;
  top: 10px;
}
</style>
