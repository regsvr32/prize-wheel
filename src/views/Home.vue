<template>
  <div class="main-container">
    <div class="container-left">
      <div class="wheel-area">
        <div class="wheel-wrapper">
          <canvas ref="wheelCanvas" width="1000" height="1000" :style="`transform: rotate(${wheelRotate}turn)`" />
          <svg class="pointer" width="100" height="32" viewBox="0 0 100 32" preserveAspectRatio="none">
            <polygon v-if="prizeList.length" points="0,16 100,0 100,32" style="fill:#ffffff;stroke:#000000;stroke-width:1" />
          </svg>
        </div>
        <div class="dashboard">
          <h2>{{pointingPrize}}</h2>
          <div v-if="prizeList.length" class="button-container">
            <button class="start-roll" @click="startRoll" :disabled="rolling">&nbsp;转起！</button>
          </div>
        </div>
        <div class="log-action">
          <button @click="clearLog">清空记录</button>
        </div>
      </div>
      <div class="roll-log">
        {{rollRecordLog}}
      </div>
    </div>
    <div class="prize-list">
      <h4>转盘设置：</h4>
      <div v-if="depositKey" class="prize-action">
        <button @click="loadDeposit" :disabled="rolling">重置为预设</button>
      </div>
      <textarea :readonly="rolling" v-model="rawPrizes" ref="prizeInput" />
      <div class="github-link">
        <a target="_blank" href="https://github.com/regsvr32/prize-wheel">
          <img :src="gitHubIcon" /> GitHub
        </a>
      </div>
    </div>
  </div>
</template>

<style lang="sass">
*
  box-sizing: border-box
body
  margin: 0px
.main-container
  width: 100vw
  height: 100vh
  display: flex
  .container-left
    display: flex
    width: calc(100% - 360px)
    flex-direction: column
    border-right: 1px solid #cccccc
    .wheel-area
      position: relative
      padding: 20px 40px
      border-bottom: 1px solid #cccccc
      width: 100%
      height: calc(100% - 300px)
      display: flex
      .wheel-wrapper
        position: relative
        max-width: 100%
        max-height: 100%
        aspect-ratio: 1 / 1
        canvas
          width: 100%
          height: 100%
        .pointer
          position: absolute
          right: -1%
          top: 48.4%
          height: 3.2%
          width: 20%
      .dashboard
        flex-grow: 1
        padding-left: 40px
        text-align: center
        overflow: auto
        h2
          color: #2acb2a
        .button-container
          margin: 100px 0px 20px
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
      .log-action
        position: absolute
        bottom: 12px
        right: 12px
    .roll-log
      position: relative
      flex-grow: 1
      overflow: auto
      padding: 20px
      white-space: pre-wrap
  .prize-list
    position: relative
    flex-grow: 1
    padding: 20px
    h4
      margin: 0px 0px 12px
    .prize-action
      position: absolute
      top: 18px
      right: 20px
    textarea
      width: 100%
      resize: none
      font-size: 14px
      height: calc(100% - 60px)
      border: none
      &:focus
        outline-color: #719ece
    .github-link
      margin-top: 8px 
      text-align: end
      a
        text-decoration: none
        color: #0969da
        img
          vertical-align: text-top
          height: 1.2em
</style>

<script setup>
import { ref, computed, watch, watchEffect, nextTick } from 'vue'
import { useRoute } from 'vue-router'
import { debounce } from 'lodash'
import gitHubIcon from '../assets/GitHub-Mark-32px.png'

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
        if (!prizeInput.value) { return }
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
    ctx.font = '40px sans'
    const measure = ctx.measureText(prize)
    const x = 270 - measure.width / 2
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
  rotateTurn = 9 + Math.random() * 3 - rotateFrom
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
