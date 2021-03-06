<template>
  <div class="main-container" :class="{ portrait }" :style="{ height: windowHeight }">
    <div class="container-left">
      <h1 class="pointing-prize">{{pointingPrize}}</h1>
      <div class="wheel-area">
        <div class="wheel-wrapper">
          <canvas ref="wheelCanvas" width="1000" height="1000" :style="`transform: rotate(${wheelRotate}turn)`" />
          <svg class="pointer" width="100" height="32" viewBox="0 0 100 32" preserveAspectRatio="none">
            <polygon v-if="prizeList.length" points="0,16 100,0 100,32" style="fill:#ffffff;stroke:#000000;stroke-width:1" />
          </svg>
        </div>
      </div>
      <div v-if="prizeList.length" class="button-container">
        <button class="start-roll" @click="startRoll" :disabled="rolling">&nbsp;转起！</button>
      </div>
    </div>
    <div class="container-right" :class="{ opened: drawerOpened }">
      <div class="prize-list">
        <h4>转盘设置：</h4>
        <textarea :readonly="rolling" v-model="rawPrizes" ref="prizeInput" />
      </div>
      <div class="roll-log">
        {{rollRecordLog}}
      </div>
      <div class="footer">
        <button v-if="depositKey" @click="loadDeposit" :disabled="rolling">重置为预设</button>
        <button @click="clearLog">清空记录</button>
        <span style="flex-grow: 1" />
        <a class="github-link" target="_blank" href="https://github.com/regsvr32/prize-wheel">
          <img :src="gitHubIcon" /> GitHub
        </a>
      </div>
    </div>
    <div v-if="portrait" class="switch-drawer" @click="drawerOpened = !drawerOpened">{{drawerOpened ? '❌' : '🔧'}}</div>
  </div>
</template>

<style lang="sass">
*
  box-sizing: border-box
body
  margin: 0px
  overflow: hidden
.main-container
  width: 100vw
  display: flex
  overflow: hidden
  .container-left
    display: flex
    flex-direction: column
    flex-grow: 1
    .pointing-prize
      text-align: center
      color: #2acb2a
    .wheel-area
      width: 100%
      padding: 40px
      height: calc(100% - 270px)
      .wheel-wrapper
        position: relative
        max-width: 100%
        max-height: 100%
        margin: auto
        aspect-ratio: 1 / 1
        pointer-events: none
        canvas
          width: 100%
          height: 100%
        .pointer
          position: absolute
          right: -1%
          top: 48.4%
          height: 3.2%
          width: 20%
    .button-container
      flex-grow: 1
      padding-top: 60px
      text-align: center
      .start-roll
        font-size: 24px
        padding: 10px 30px
        background-color: #719ece
        border: 1px solid #666666
        color: #3b2b55
        border-radius: 4px 
        &:hover
          background-color: #b3cbe5
          color: #60468b
          transform: scale(1.1)
        &:active
          transform: unset
        &:disabled
          background-color: #eeeeee
          color: #777777
          transform: unset
  .container-right
    display: flex
    flex-direction: column
    width: 25%
    min-width: 360px
    border-left: 1px solid #cccccc
    background-color: #ffffff
    .prize-list
      display: flex
      flex-direction: column
      padding: 20px
      flex-grow: 1
      border-bottom: 1px solid #cccccc
      h4
        margin: 0px 0px 12px
      textarea
        flex: 1
        width: 100%
        resize: none
        font-size: 14px
        border: none
        &:focus
          outline-color: #719ece
    .roll-log
      padding: 20px
      flex: 1
      overflow: auto
      white-space: pre-wrap
      border-bottom: 1px solid #cccccc
    .footer
      display: flex
      padding: 12px
      &>*
        margin-right: 12px
      .github-link
        text-decoration: none
        color: #0969da
        img
          vertical-align: text-top
          height: 1.2em
  .switch-drawer
    position: absolute
    top: 12px
    right: 12px
    user-select: none
    filter: grayscale(100%)
    opacity: 0.5
  &.portrait
    display: block
    .container-left
      width: 100%
      height: 100%
      .wheel-area
        padding: 32px 16px
        height: calc(100% - 210px)
      .button-container
        padding-top: 24px
    .container-right
      position: absolute
      width: 96%
      min-width: unset
      height: 100%
      top: 0%
      left: 100%
      transition: left 0.2s ease
      &.opened
        left: 4%
    .prize-list, .roll-log
      padding: 12px
</style>

<script setup>
import { ref, computed, watch, watchEffect, nextTick, onMounted, onUnmounted } from 'vue'
import { useRoute } from 'vue-router'
import { debounce } from 'lodash'
import gitHubIcon from '../assets/GitHub-Mark-32px.png'

const portrait = ref(undefined)
const drawerOpened = ref(false)
const windowHeight = ref('100vh')

function fitOrientation() {
  portrait.value = window.matchMedia("(orientation: portrait)").matches
  windowHeight.value = `${window.innerHeight}px`
}

onMounted(() => {
  window.addEventListener("resize", fitOrientation)
  fitOrientation()
})
onUnmounted(() => {
  window.removeEventListener("resize", fitOrientation)
})

const wheelCanvas = ref(null)
const rawPrizes = ref('')
const prizeList = computed(() => rawPrizes.value.split('\n').filter(p => /\S/.test(p)))

const route = useRoute()
const { depositKey } = route.params

watch(rawPrizes, debounce(value => {
  localStorage.setItem(`prizes_${depositKey}`, value)
}, 800))

rawPrizes.value = localStorage.getItem(`prizes_${depositKey}`) || ''

async function loadDeposit() {
  if (!depositKey) { return }
  const resp = await fetch(`/deposit/${depositKey}.txt`)
  if (resp.ok && /^text\/plain/.test(resp.headers.get('Content-Type'))) {
    rawPrizes.value = await resp.text()
  }
}

const prizeInput = ref(null)
if (!rawPrizes.value) {
  loadDeposit().then(() => {
    if (!prizeList.value.length) {
      const stopHandle = watchEffect(() => {
        if (!prizeInput.value || portrait.value === undefined) { return }
        if (portrait.value) {
          drawerOpened.value = true
        }
        prizeInput.value.focus()
        nextTick(stopHandle)
      })
    }
  })
}

watchEffect(() => {
  if (!wheelCanvas.value) { return }
  /** @type {CanvasRenderingContext2D} */
  const ctx = wheelCanvas.value.getContext('2d')
  ctx.clearRect(0, 0, 1000, 1000)
  const length = prizeList.value.length
  if (!length) { return }
  let angleFrom = 0, angleTo = 0
  prizeList.value.forEach((prize, idx) => {
    // pie
    ctx.fillStyle = `hsl(${(idx / length) * 320},85%,60%)`
    ctx.beginPath()
    ctx.moveTo(500, 500)
    angleFrom = angleTo
    angleTo += Math.PI * 2 / length
    ctx.arc(500, 500, 500, angleFrom, angleTo)
    ctx.lineTo(500, 500);
    ctx.fill()
    // label
    ctx.save()
    ctx.translate(500, 500)
    ctx.rotate((angleFrom + angleTo) / 2)
    ctx.fillStyle = '#ffffff'
    ctx.strokeStyle = '#000000'
    ctx.font = `${portrait.value ? 72 : 48}px sans-serif`
    const measure = ctx.measureText(prize)
    const x = 280 - measure.width / 2
    const y = measure.actualBoundingBoxAscent / 2
    ctx.fillText(prize, x, y)
    ctx.strokeText(prize, x, y)
    ctx.restore()
  })
})

const wheelRotate = ref(0)
const pointingPrize = computed(() => {
  const length = prizeList.value.length
  if (!length) { return '' }
  return prizeList.value[(length - Math.ceil(wheelRotate.value % 1 * length)) % length]
})

const rolling = ref(false)

function easeOutCubic(x) {
  return 1 - Math.pow(1 - x, 3)
}

let rotateFrom = 0
let rotateTurn = 0
let startTime = 0
function rollTick(time) {
  if (!startTime && time) {
    startTime = time
  }
  const timeRate = Math.min(1, (time - startTime) / 5000)
  wheelRotate.value = rotateFrom + rotateTurn * easeOutCubic(timeRate)
  if (timeRate < 1) {
    requestAnimationFrame(rollTick)
  }
  else {
    handleRollResult()
  }
}

function startRoll() {
  rolling.value = true
  rotateFrom = wheelRotate.value % 1
  const arr = new Uint16Array(1)
  window.crypto.getRandomValues(arr)
  rotateTurn = 9 + arr[0] / 0x10000 * 3 - rotateFrom
  startTime = 0
  rollTick(0)
}

const rollRecordLog = ref('')
function formatRecord({ time, result }) {
  return `[${time.toLocaleString('default', { hour12: false })}] ${result}\n`
}

let db = null
const dbOpenRequest = indexedDB.open('prize_wheel')
dbOpenRequest.onupgradeneeded = ({ target: { result } }) => {
  result.createObjectStore('roll_record', { autoIncrement: true })
}
dbOpenRequest.onsuccess = () => {
  db = dbOpenRequest.result
  const request = db.transaction('roll_record').objectStore('roll_record').getAll()
  request.onsuccess = () => {
    rollRecordLog.value = request.result.map(formatRecord).join('')
  }
}

function handleRollResult() {
  const record = { time: new Date(), result: pointingPrize.value }
  if (db) {
    db.transaction('roll_record', 'readwrite').objectStore('roll_record').put(record)
  }
  rollRecordLog.value += formatRecord(record)
  rolling.value = false
}

function clearLog() {
  if (!window.confirm('确定清除所有记录？')) { return }
  rollRecordLog.value = ''
  if (db) {
    db.transaction('roll_record', 'readwrite').objectStore('roll_record').clear()
  }
}
</script>
