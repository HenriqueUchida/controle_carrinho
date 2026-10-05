<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue'

/*
  Protocolo (descoberto no Wireshark), texto ASCII via Write Command:
    "velocidade,ângulo"   velocidade 0–100, ângulo 0–360 (90 = frente, 270 = trás)
    "0,0"                 parada
  Confirmados: 100,90 (frente), 100,270 (trás) e 100,360 (direita).
  Suposto (ainda não validado): 100,180 = esquerda.

  Dois joysticks: o da esquerda controla avanço/recuo (eixo Y) e o da direita controla
  a direção (eixo X). Os dois eixos são combinados em um único vetor e convertidos para
  velocidade + ângulo, que é o que o carrinho entende.
*/

// ---------- Créditos ----------
const CREDIT_NAME = 'Henrique Uchida'
const CREDIT_URL = 'https://github.com/HenriqueUchida'
const LOGO_URL = `${import.meta.env.BASE_URL}senac-logo.png` // coloque o arquivo em public/

// ---------- Ajustes ----------
const SEND_EVERY_MS = 50 // o app oficial manda dezenas de pacotes por segundo
const STOP_REPEATS = 2 // o app enviou "0,0" duas vezes ao soltar o joystick
const DEADZONE = 5 // abaixo disso (em %) consideramos o joystick solto
const TRAVEL_THROTTLE = 60 // curso do botão (px) no joystick vertical
const TRAVEL_STEER = 100 // curso do botão (px) no joystick horizontal

// ---------- UUIDs do carrinho (lidos no nRF Connect) ----------
const SERVICE_UUID = 'dacabf1f-5f2e-4d16-b8f8-13bbaaec1349'
const CHARACTERISTIC_UUID = 'dacabf1f-5f2e-4d16-b8f8-13bbaaec5781' // Write Without Response

// ---------- Estado ----------
const status = ref('idle') // idle | connecting | connected
const deviceName = ref('')
const error = ref('')
const lastSent = ref('—')
const battery = ref(null)
const logoOk = ref(true)
const isFullscreen = ref(false)

const padThrottle = ref(0) // -1 (trás) .. 1 (frente)
const padSteer = ref(0) // -1 (esquerda) .. 1 (direita)
const keyThrottle = ref(0)
const keySteer = ref(0)

const supported = typeof navigator !== 'undefined' && 'bluetooth' in navigator
const canFullscreen = typeof document !== 'undefined' && document.fullscreenEnabled
const connected = computed(() => status.value === 'connected')
const knobY = computed(() => -(padThrottle.value || keyThrottle.value) * TRAVEL_THROTTLE)
const knobX = computed(() => (padSteer.value || keySteer.value) * TRAVEL_STEER)

let device = null
let characteristic = null
let command = '0,0'
let stopBurst = 0
let writing = false
let timer = null
const encoder = new TextEncoder()

// ---------- Tela cheia + horizontal ----------
// Só funciona com um toque do usuário, e nem todo navegador permite travar a rotação.
// Se não der, o aviso de "gire o celular" cobre a tela em modo retrato.
async function enterLandscape() {
  try {
    if (!document.fullscreenElement) await document.documentElement.requestFullscreen()
    await screen.orientation?.lock?.('landscape')
  } catch {
    /* sem permissão: segue */
  }
}
const onFullscreenChange = () => (isFullscreen.value = !!document.fullscreenElement)

// ---------- Conexão ----------
async function connect() {
  error.value = ''
  status.value = 'connecting'
  enterLandscape() // sem await, para não perder o toque que autoriza o Bluetooth
  try {
    // Sem filtro por nome: o nome pode ser alterado no app da RoboCore.
    device = await navigator.bluetooth.requestDevice({
      acceptAllDevices: true,
      optionalServices: [SERVICE_UUID, 'battery_service'],
    })
    device.addEventListener('gattserverdisconnected', onDisconnected)
    deviceName.value = device.name || 'Dispositivo sem nome'

    const server = await device.gatt.connect()
    const svc = await server.getPrimaryService(SERVICE_UUID)
    characteristic = await svc.getCharacteristic(CHARACTERISTIC_UUID)

    // Sequência observada no app oficial: assina a bateria e manda "0,0" antes de mover
    await subscribeBattery(server)
    await write('0,0')
    await write('0,0')

    status.value = 'connected'
    startLoop()
  } catch (e) {
    status.value = 'idle'
    if (e.name === 'NotFoundError' && /cancel/i.test(e.message)) return
    error.value = explain(e)
  }
}

// O firmware só aceita movimento depois que o cliente liga as notificações de bateria.
async function subscribeBattery(server) {
  try {
    const svc = await server.getPrimaryService('battery_service')
    try {
      const level = await svc.getCharacteristic('battery_level')
      level.addEventListener('characteristicvaluechanged', (e) => {
        battery.value = e.target.value.getUint8(0)
      })
      await level.startNotifications()
    } catch {
      /* sem Battery Level: segue */
    }
    try {
      const levelStatus = await svc.getCharacteristic(0x2bed) // Battery Level Status
      await levelStatus.startNotifications()
    } catch {
      /* sem Battery Level Status: segue */
    }
  } catch {
    /* sem serviço de bateria: segue */
  }
}

function explain(e) {
  const msg = e?.message || String(e)
  if (e?.name === 'NotFoundError' && /service/i.test(msg)) {
    return 'Serviço não encontrado. Este dispositivo não parece ser um Hockey Bot compatível.'
  }
  if (e?.name === 'NotFoundError' && /characteristic/i.test(msg)) {
    return 'Característica de escrita não encontrada no carrinho.'
  }
  if (e?.name === 'SecurityError') return 'A Web Bluetooth exige HTTPS (ou localhost).'
  return msg
}

function disconnect() {
  if (device?.gatt?.connected) device.gatt.disconnect()
}

function onDisconnected() {
  stopLoop()
  characteristic = null
  battery.value = null
  status.value = 'idle'
  resetControls()
}

// ---------- Envio ----------
function startLoop() {
  stopLoop()
  timer = setInterval(tick, SEND_EVERY_MS)
}
function stopLoop() {
  clearInterval(timer)
  timer = null
}

function tick() {
  if (command !== '0,0') return write(command)
  if (stopBurst > 0) {
    stopBurst--
    write('0,0')
  }
}

async function write(text) {
  if (!characteristic || writing) return
  writing = true
  try {
    const data = encoder.encode(text)
    // O app oficial usa "Write Command" (sem resposta), que é mais rápido
    if (characteristic.properties.writeWithoutResponse) {
      await characteristic.writeValueWithoutResponse(data)
    } else {
      await characteristic.writeValue(data)
    }
    lastSent.value = text
  } catch (e) {
    error.value = explain(e)
  } finally {
    writing = false
  }
}

function setCommand(speed, angle) {
  if (speed < DEADZONE) {
    if (command !== '0,0') stopBurst = STOP_REPEATS
    command = '0,0'
  } else {
    command = `${speed},${angle}`
  }
}

// Combina avanço (Y) e direção (X) em velocidade + ângulo
function updateCommand() {
  const t = padThrottle.value || keyThrottle.value
  const s = padSteer.value || keySteer.value
  const speed = Math.round(Math.min(1, Math.hypot(s, t)) * 100)
  let angle = Math.round(((Math.atan2(t, s) * 180) / Math.PI + 360) % 360)
  if (angle === 0) angle = 360 // o app oficial manda 360 para a direita, nunca 0
  setCommand(speed, angle)
}

function resetControls() {
  padThrottle.value = 0
  padSteer.value = 0
  keyThrottle.value = 0
  keySteer.value = 0
  keys.clear()
  updateCommand()
}

// ---------- Joysticks ----------
const clamp = (v) => Math.max(-1, Math.min(1, v))

function onPadDown(e, axis) {
  if (!connected.value) return
  e.currentTarget.setPointerCapture(e.pointerId)
  onPadMove(e, axis)
}

function onPadMove(e, axis) {
  if (!connected.value || !e.currentTarget.hasPointerCapture(e.pointerId)) return
  const r = e.currentTarget.getBoundingClientRect()
  if (axis === 'throttle') {
    padThrottle.value = clamp(-(e.clientY - (r.top + r.height / 2)) / TRAVEL_THROTTLE)
  } else {
    padSteer.value = clamp((e.clientX - (r.left + r.width / 2)) / TRAVEL_STEER)
  }
  updateCommand()
}

function onPadUp(axis) {
  if (axis === 'throttle') padThrottle.value = 0
  else padSteer.value = 0
  updateCommand()
}

// ---------- Teclado (WASD / setas) ----------
const keys = new Set()
const KEYMAP = {
  w: 'up', arrowup: 'up',
  s: 'down', arrowdown: 'down',
  a: 'left', arrowleft: 'left',
  d: 'right', arrowright: 'right',
}

function onKey(e, down) {
  const dir = KEYMAP[e.key.toLowerCase()]
  if (!dir) return
  e.preventDefault()
  if (down) keys.add(dir)
  else keys.delete(dir)
  keyThrottle.value = (keys.has('up') ? 1 : 0) - (keys.has('down') ? 1 : 0)
  keySteer.value = (keys.has('right') ? 1 : 0) - (keys.has('left') ? 1 : 0)
  if (connected.value) updateCommand()
}
const onKeyDown = (e) => !e.repeat && onKey(e, true)
const onKeyUp = (e) => onKey(e, false)

onMounted(() => {
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
  document.addEventListener('fullscreenchange', onFullscreenChange)
})
onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
  document.removeEventListener('fullscreenchange', onFullscreenChange)
  stopLoop()
  disconnect()
})
</script>

<template>
  <main class="fixed inset-0 grid select-none grid-rows-[auto_1fr_auto] overflow-hidden bg-[#EAF2F6] text-[#14212B]">
    <!-- Cabeçalho: nome, status e botões -->
    <header class="flex items-center justify-between gap-4 border-b border-[#004A8D]/15 bg-white px-4 py-2">
      <div class="min-w-0">
        <h1 class="text-lg font-bold leading-tight">Hockey Bot</h1>
        <p class="truncate text-xs" :class="connected ? 'text-[#004A8D]' : 'text-[#5B6B77]'">
          <template v-if="connected">
            Conectado a {{ deviceName }}<span v-if="battery !== null"> · bateria {{ battery }}%</span>
          </template>
          <template v-else-if="status === 'connecting'">Conectando…</template>
          <template v-else>Desconectado</template>
        </p>
      </div>

      <div class="flex shrink-0 gap-2">
        <button
          v-if="!connected"
          class="rounded-full bg-[#004A8D] px-5 py-1.5 text-sm font-semibold text-white focus:outline-none focus-visible:ring-4 focus-visible:ring-[#004A8D]/40 disabled:opacity-50"
          :disabled="!supported || status === 'connecting'"
          @click="connect"
        >
          Conectar
        </button>
        <button
          v-else
          class="rounded-full border-2 border-[#004A8D] px-5 py-1 text-sm font-semibold text-[#004A8D] focus:outline-none focus-visible:ring-4 focus-visible:ring-[#004A8D]/30"
          @click="disconnect"
        >
          Desconectar
        </button>
        <button
          class="rounded-full bg-[#C8362B] px-5 py-1.5 text-sm font-bold text-white focus:outline-none focus-visible:ring-4 focus-visible:ring-[#C8362B]/40 disabled:opacity-40"
          :disabled="!connected"
          @click="resetControls"
        >
          Parar
        </button>
      </div>
    </header>

    <!-- Corpo: joystick, logo no centro, joystick -->
    <div class="grid grid-cols-[auto_1fr_auto] items-center gap-4 px-6">
      <!-- Joystick esquerdo: avançar e recuar -->
      <div
        class="relative h-52 w-24 touch-none rounded-full border-4 border-[#004A8D] bg-white transition-opacity"
        :class="connected ? 'opacity-100' : 'opacity-50'"
        role="slider"
        aria-label="Avançar e recuar"
        @pointerdown="onPadDown($event, 'throttle')"
        @pointermove="onPadMove($event, 'throttle')"
        @pointerup="onPadUp('throttle')"
        @pointercancel="onPadUp('throttle')"
      >
        <div class="absolute inset-x-0 top-1/2 h-0.5 -translate-y-1/2 bg-[#C8362B]/60"></div>
        <div
          class="absolute left-1/2 top-1/2 h-20 w-20 rounded-full bg-[#14212B] shadow-md"
          :style="{ transform: `translate(-50%, calc(-50% + ${knobY}px))` }"
        ></div>
      </div>

      <!-- Centro: logo do Senac -->
      <section class="flex min-w-0 flex-col items-center gap-3 text-center">
        <img v-if="logoOk" :src="LOGO_URL" alt="Senac" class="h-28 max-w-full object-contain" @error="logoOk = false" />
        <span v-else class="text-5xl font-bold text-[#004A8D]">Senac</span>

        <p v-if="!supported" class="max-w-xs rounded-lg bg-[#C8362B]/10 p-2 text-xs text-[#8F2219]">
          Este navegador não suporta Web Bluetooth. Use o Chrome no Android.
        </p>
        <p v-if="error" class="max-w-xs rounded-lg bg-[#C8362B]/10 p-2 text-xs text-[#8F2219]" role="alert">
          {{ error }}
        </p>

        <button v-if="canFullscreen && !isFullscreen" class="text-xs text-[#004A8D] underline" @click="enterLandscape">
          Tela cheia
        </button>
        <p class="text-xs text-[#5B6B77]">
          Último comando:
          <code class="rounded bg-white px-1.5 py-0.5 font-mono text-[#14212B]">{{ lastSent }}</code>
        </p>
      </section>

      <!-- Joystick direito: virar -->
      <div
        class="relative h-24 w-72 touch-none rounded-full border-4 border-[#004A8D] bg-white transition-opacity"
        :class="connected ? 'opacity-100' : 'opacity-50'"
        role="slider"
        aria-label="Virar para a esquerda ou direita"
        @pointerdown="onPadDown($event, 'steer')"
        @pointermove="onPadMove($event, 'steer')"
        @pointerup="onPadUp('steer')"
        @pointercancel="onPadUp('steer')"
      >
        <div class="absolute inset-y-0 left-1/2 w-0.5 -translate-x-1/2 bg-[#C8362B]/60"></div>
        <div
          class="absolute left-1/2 top-1/2 h-20 w-20 rounded-full bg-[#14212B] shadow-md"
          :style="{ transform: `translate(calc(-50% + ${knobX}px), -50%)` }"
        ></div>
      </div>
    </div>

    <!-- Créditos -->
    <p class="py-2 text-center text-xs text-[#5B6B77]">
      Desenvolvido por
      <a :href="CREDIT_URL" target="_blank" rel="noopener" class="font-semibold text-[#004A8D] underline">{{ CREDIT_NAME }}</a>
    </p>

    <!-- Aviso em modo retrato -->
    <div
      class="fixed inset-0 z-50 hidden flex-col items-center justify-center gap-3 bg-[#004A8D] p-8 text-center text-white portrait:max-md:flex"
    >
      <p class="text-2xl font-bold">Gire o celular</p>
      <p class="text-sm">O controle funciona na horizontal, com um joystick em cada mão.</p>
    </div>
  </main>
</template>
