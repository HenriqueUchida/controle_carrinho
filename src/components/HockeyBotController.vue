<script setup>
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue'

/*
  Protocolo descoberto no Wireshark (texto ASCII enviado via Write Command):
    "velocidade,ângulo"   velocidade 0–100, ângulo 0–360 (90 = frente, 270 = trás)
    "0,0"                 parada
  Confirmados: 100,90 (frente) e 100,270 (trás).
  Supostos (ainda não capturados): 100,180 (esquerda) e 100,0 (direita).
*/

const SEND_EVERY_MS = 50 // o app oficial manda dezenas de pacotes por segundo
const STOP_REPEATS = 2 // o app enviou "0,0" duas vezes ao soltar o joystick
const DEADZONE = 5 // abaixo disso (em %) consideramos o joystick solto

// ---------- UUIDs do carrinho (lidos no nRF Connect) ----------
const SERVICE_UUID = 'dacabf1f-5f2e-4d16-b8f8-13bbaaec1349'
const CHARACTERISTIC_UUID = 'dacabf1f-5f2e-4d16-b8f8-13bbaaec5781' // Write Without Response

// ---------- Estado ----------
const status = ref('idle') // idle | connecting | connected
const deviceName = ref('')
const error = ref('')
const lastSent = ref('—')
const knob = reactive({ x: 0, y: 0 })
const pad = ref(null)

const supported = typeof navigator !== 'undefined' && 'bluetooth' in navigator
const connected = computed(() => status.value === 'connected')

let device = null
let characteristic = null
let command = '0,0'
let stopBurst = 0
let writing = false
let timer = null
const encoder = new TextEncoder()

// ---------- Conexão ----------
async function connect() {
  error.value = ''
  status.value = 'connecting'
  try {
    // Sem filtro por nome: o nome pode ser alterado no app da RoboCore.
    // O usuário escolhe o carrinho na lista, e só ele precisa ter o serviço abaixo.
    device = await navigator.bluetooth.requestDevice({
      acceptAllDevices: true,
      optionalServices: [SERVICE_UUID],
    })
    device.addEventListener('gattserverdisconnected', onDisconnected)
    deviceName.value = device.name || 'Dispositivo sem nome'

    const server = await device.gatt.connect()
    const svc = await server.getPrimaryService(SERVICE_UUID)
    characteristic = await svc.getCharacteristic(CHARACTERISTIC_UUID)

    status.value = 'connected'
    startLoop()
  } catch (e) {
    status.value = 'idle'
    if (e.name === 'NotFoundError' && /cancel/i.test(e.message)) return
    error.value = explain(e)
  }
}

function explain(e) {
  const msg = e?.message || String(e)
  if (e?.name === 'NotFoundError' && /service/i.test(msg)) {
    return 'Serviço não encontrado. Este carrinho não parece ser um Hockey Bot compatível.'
  }
  if (e?.name === 'NotFoundError' && /characteristic/i.test(msg)) {
    return 'Característica de escrita não encontrada no carrinho.'
  }
  if (e?.name === 'SecurityError') {
    return 'A Web Bluetooth exige HTTPS (ou localhost).'
  }
  return msg
}

function disconnect() {
  if (device?.gatt?.connected) device.gatt.disconnect()
}

function onDisconnected() {
  stopLoop()
  characteristic = null
  status.value = 'idle'
  resetKnob()
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

// ---------- Joystick virtual ----------
function onPointerDown(e) {
  if (!connected.value) return
  e.currentTarget.setPointerCapture(e.pointerId)
  onPointerMove(e)
}

function onPointerMove(e) {
  if (!connected.value || !(e.buttons & 1 || e.pointerType === 'touch')) return
  const rect = pad.value.getBoundingClientRect()
  const maxR = rect.width / 2 - 48 // 48 = metade do botão
  let dx = e.clientX - (rect.left + rect.width / 2)
  let dy = e.clientY - (rect.top + rect.height / 2)
  const dist = Math.hypot(dx, dy)
  if (dist > maxR) {
    dx = (dx / dist) * maxR
    dy = (dy / dist) * maxR
  }
  knob.x = dx
  knob.y = dy

  const speed = Math.round((Math.min(dist, maxR) / maxR) * 100)
  const angle = Math.round(((Math.atan2(-dy, dx) * 180) / Math.PI + 360) % 360)
  setCommand(speed, angle)
}

function onPointerUp() {
  resetKnob()
}

function resetKnob() {
  knob.x = 0
  knob.y = 0
  setCommand(0, 0)
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
  if (e.target.tagName === 'INPUT') return
  const dir = KEYMAP[e.key.toLowerCase()]
  if (!dir) return
  e.preventDefault()
  if (down) keys.add(dir)
  else keys.delete(dir)
  if (!connected.value) return

  const x = (keys.has('right') ? 1 : 0) - (keys.has('left') ? 1 : 0)
  const y = (keys.has('up') ? 1 : 0) - (keys.has('down') ? 1 : 0)
  if (x === 0 && y === 0) {
    resetKnob()
    return
  }
  const angle = Math.round(((Math.atan2(y, x) * 180) / Math.PI + 360) % 360)
  knob.x = x * 70
  knob.y = -y * 70
  setCommand(100, angle)
}
const onKeyDown = (e) => !e.repeat && onKey(e, true)
const onKeyUp = (e) => onKey(e, false)

onMounted(() => {
  window.addEventListener('keydown', onKeyDown)
  window.addEventListener('keyup', onKeyUp)
})
onBeforeUnmount(() => {
  window.removeEventListener('keydown', onKeyDown)
  window.removeEventListener('keyup', onKeyUp)
  stopLoop()
  disconnect()
})
</script>

<template>
  <main class="min-h-screen bg-[#EAF2F6] text-[#14212B] flex flex-col items-center gap-6 px-4 py-8">
    <header class="w-full max-w-sm flex items-center justify-between">
      <div>
        <h1 class="text-2xl font-bold">Hockey Bot</h1>
        <p class="text-sm" :class="connected ? 'text-[#1F5FA8]' : 'text-[#5B6B77]'">
          <template v-if="connected">Conectado a {{ deviceName }}</template>
          <template v-else-if="status === 'connecting'">Conectando…</template>
          <template v-else>Desconectado</template>
        </p>
      </div>
      <button
        v-if="!connected"
        class="rounded-full bg-[#1F5FA8] px-5 py-2.5 font-semibold text-white disabled:opacity-50 focus:outline-none focus-visible:ring-4 focus-visible:ring-[#1F5FA8]/40"
        :disabled="!supported || status === 'connecting'"
        @click="connect"
      >
        Conectar
      </button>
      <button
        v-else
        class="rounded-full border-2 border-[#C8362B] px-5 py-2 font-semibold text-[#C8362B] focus:outline-none focus-visible:ring-4 focus-visible:ring-[#C8362B]/30"
        @click="disconnect"
      >
        Desconectar
      </button>
    </header>

    <p v-if="!supported" class="w-full max-w-sm rounded-lg bg-[#C8362B]/10 p-3 text-sm text-[#8F2219]">
      Este navegador não suporta Web Bluetooth. Use o Chrome ou o Edge no Android ou no computador.
      No iPhone não funciona, nem no Chrome.
    </p>
    <p v-if="error" class="w-full max-w-sm rounded-lg bg-[#C8362B]/10 p-3 text-sm text-[#8F2219]" role="alert">
      {{ error }}
    </p>

    <!-- Joystick: um rinque com o botão (puck) no centro -->
    <div
      ref="pad"
      class="relative h-72 w-72 touch-none select-none rounded-full border-4 border-[#1F5FA8] bg-white transition-opacity"
      :class="connected ? 'opacity-100' : 'opacity-50'"
      @pointerdown="onPointerDown"
      @pointermove="onPointerMove"
      @pointerup="onPointerUp"
      @pointercancel="onPointerUp"
    >
      <div class="absolute inset-x-0 top-1/2 h-1 -translate-y-1/2 bg-[#C8362B]/70"></div>
      <div class="absolute left-1/2 top-1/2 h-28 w-28 -translate-x-1/2 -translate-y-1/2 rounded-full border-2 border-[#1F5FA8]/40"></div>
      <div
        class="absolute left-1/2 top-1/2 h-24 w-24 rounded-full bg-[#14212B] shadow-md"
        :style="{ transform: `translate(calc(-50% + ${knob.x}px), calc(-50% + ${knob.y}px))` }"
      ></div>
    </div>

    <button
      class="w-72 rounded-full bg-[#C8362B] py-3 text-lg font-bold text-white disabled:opacity-40 focus:outline-none focus-visible:ring-4 focus-visible:ring-[#C8362B]/40"
      :disabled="!connected"
      @click="resetKnob"
    >
      Parar
    </button>

    <p class="text-sm text-[#5B6B77]">
      Último comando enviado:
      <code class="rounded bg-white px-2 py-0.5 font-mono text-[#14212B]">{{ lastSent }}</code>
    </p>
    <p class="max-w-sm text-center text-xs text-[#5B6B77]">
      No computador, use W A S D ou as setas do teclado.
    </p>
  </main>
</template>
