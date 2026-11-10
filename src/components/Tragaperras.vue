<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, ref } from 'vue'
import { useDineroStore } from '../stores/dinero'
import { TIPOS_FICHA } from '../utils/fichas'

interface Simbolo {
  icono: string
  peso: number
  premio: number
}

const SIMBOLOS: Simbolo[] = [
  { icono: '🍒', peso: 30, premio: 5 },
  { icono: '🍋', peso: 25, premio: 8 },
  { icono: '🔔', peso: 18, premio: 15 },
  { icono: '⭐', peso: 12, premio: 25 },
  { icono: '💎', peso: 8, premio: 50 },
  { icono: '7️⃣', peso: 4, premio: 100 }
]
const CEREZA = '🍒'
const PREMIO_DOS_CEREZAS = 2
const PESO_TOTAL = SIMBOLOS.reduce((total, s) => total + s.peso, 0)

const FICHAS_APUESTA = [...TIPOS_FICHA].reverse().filter((t) => t.valor <= 100)

const NUM_CARRETES = 3
const RECORRIDO = 100

const banca = useDineroStore()

interface Carrete {
  tira: string[]
  desplaza: number
  animando: boolean
  duracion: number
}

const apuesta = ref(0)
const ultimaApuesta = ref(0)
const premio = ref(0)
const gana = ref(false)
const girando = ref(false)
const mensaje = ref('Mete dinero y tira de la palanca.')

const carretes = ref<Carrete[]>(
  Array.from({ length: NUM_CARRETES }, () => ({
    tira: [simboloAleatorio(), simboloAleatorio(), simboloAleatorio()],
    desplaza: 0,
    animando: false,
    duracion: 0
  }))
)

const tablaPremios = computed(() =>
  SIMBOLOS.map((s) => ({ icono: s.icono, premio: s.premio })).reverse()
)

function simboloAleatorio(): string {
  let r = Math.random() * PESO_TOTAL
  for (const s of SIMBOLOS) {
    r -= s.peso
    if (r < 0) return s.icono
  }
  return SIMBOLOS[0]!.icono
}

const esperar = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms))
const siguienteFrame = () =>
  new Promise<void>((resolve) => requestAnimationFrame(() => requestAnimationFrame(() => resolve())))

function meter(valor: number) {
  if (girando.value) return
  if (!banca.restar(valor)) {
    mensaje.value = 'No tienes saldo suficiente.'
    return
  }
  apuesta.value += valor
  premio.value = 0
  gana.value = false
  mensaje.value = '¡Dinero dentro! Tira de la palanca.'
}

function retirar() {
  if (girando.value || apuesta.value < 1) return
  banca.sumar(apuesta.value)
  apuesta.value = 0
  mensaje.value = 'Apuesta retirada.'
}

function repetir() {
  if (girando.value || apuesta.value > 0 || ultimaApuesta.value < 1) return
  if (!banca.restar(ultimaApuesta.value)) {
    mensaje.value = 'No tienes saldo suficiente.'
    return
  }
  apuesta.value = ultimaApuesta.value
  premio.value = 0
  gana.value = false
  mensaje.value = '¡Dinero dentro! Tira de la palanca.'
}

// Si sales de la página con dinero metido y sin girar, se te devuelve
onBeforeUnmount(() => {
  if (apuesta.value > 0 && !girando.value) banca.sumar(apuesta.value)
})

function calcularPremio(linea: string[], apostado: number): number {
  const [a, b, c] = linea
  if (a === b && b === c) {
    const simbolo = SIMBOLOS.find((s) => s.icono === a)
    return simbolo ? apostado * simbolo.premio : 0
  }
  const cerezas = linea.filter((s) => s === CEREZA).length
  if (cerezas === 2) return apostado * PREMIO_DOS_CEREZAS
  return 0
}

async function girar() {
  if (girando.value) return
  if (apuesta.value < 1) {
    mensaje.value = 'Mete dinero antes de tirar de la palanca.'
    return
  }

  girando.value = true
  gana.value = false
  premio.value = 0
  mensaje.value = '¡Girando...!'

  const apostado = apuesta.value
  ultimaApuesta.value = apostado
  apuesta.value = 0

  // Solo la línea central determina el premio.
  const linea = Array.from({ length: NUM_CARRETES }, () => simboloAleatorio())
  const finales = linea.map((s) => [simboloAleatorio(), s, simboloAleatorio()])

  carretes.value.forEach((c, i) => {
    const visibles = c.tira.slice(c.desplaza, c.desplaza + 3)
    const relleno = Array.from({ length: 30 + i * 15 }, () => simboloAleatorio())
    c.tira = [...visibles, ...relleno, ...finales[i]!]
    c.desplaza = 0
    c.animando = false
  })
  await nextTick()
  await siguienteFrame()

  await Promise.all(
    carretes.value.map(async (c, i) => {
      c.duracion = 2 + i * 1
      c.animando = true
      c.desplaza = c.tira.length - 3
      await esperar(c.duracion * 1000 + 60)
      c.tira = finales[i]!
      c.desplaza = 0
      c.animando = false
    })
  )

  const ganado = calcularPremio(linea, apostado)
  if (ganado > 0) {
    banca.sumar(ganado)
    premio.value = ganado
    gana.value = true
    mensaje.value = linea.every((s) => s === '7️⃣') ? '¡¡JACKPOT!!' : `¡Premio! Ganas ${ganado}€`
  } else {
    mensaje.value = 'Sin suerte. ¡Prueba otra vez!'
  }
  girando.value = false
}

const palancaY = ref(0)
const arrastrando = ref(false)
let inicioY = 0

function alPresionar(e: PointerEvent) {
  if (girando.value) return
  arrastrando.value = true
  inicioY = e.clientY
  ;(e.currentTarget as HTMLElement).setPointerCapture(e.pointerId)
}

function alMover(e: PointerEvent) {
  if (!arrastrando.value) return
  palancaY.value = Math.min(RECORRIDO, Math.max(0, e.clientY - inicioY))
}

async function alSoltar() {
  if (!arrastrando.value) return
  arrastrando.value = false
  const recorrido = palancaY.value

  if (recorrido >= RECORRIDO * 0.7) {
    palancaY.value = 0
    girar()
  } else if (recorrido < 6) {
    await tirarConClick()
  } else {
    palancaY.value = 0
  }
}

function alCancelar() {
  arrastrando.value = false
  palancaY.value = 0
}

async function tirarConClick() {
  if (girando.value) return
  palancaY.value = RECORRIDO
  await esperar(200)
  palancaY.value = 0
  girar()
}
</script>

<template>
  <section class="tragaperras">
    <div class="contenido">
      <div class="zona">
        <div class="maquina" :class="{ gana, girando }">
          <div class="marquesina">
            <div class="bombillas" aria-hidden="true">
              <span v-for="n in 12" :key="n" class="bombilla"></span>
            </div>
            <h2>Super 7</h2>
          </div>

          <div class="ventana">
            <div v-for="(c, i) in carretes" :key="i" class="carrete">
              <div
                class="tira"
                :class="{ animando: c.animando }"
                :style="{ '--desplaza': c.desplaza, '--dur': c.duracion + 's' }"
              >
                <span v-for="(s, j) in c.tira" :key="j" class="simbolo">{{ s }}</span>
              </div>
            </div>
            <div class="linea" aria-hidden="true"></div>
          </div>

          <div class="panel">
            <div class="pantalla">
              <span>Apuesta</span>
              <strong>{{ apuesta }}€</strong>
            </div>
            <div class="pantalla">
              <span>Premio</span>
              <strong>{{ premio }}€</strong>
            </div>
          </div>

          <p class="mensaje" aria-live="polite">{{ mensaje }}</p>

          <div class="fichas-meter" role="group" aria-label="Meter dinero">
            <button
              v-for="t in FICHAS_APUESTA"
              :key="t.valor"
              class="ficha-boton"
              :style="{ '--color': t.color, '--borde': t.borde, '--texto': t.texto }"
              :disabled="girando || t.valor > banca.dinero"
              :aria-label="`Meter ${t.valor} euros`"
              @click="meter(t.valor)"
            >
              {{ t.valor }}
            </button>
          </div>

          <div class="acciones">
            <button class="btn btn-secundario" :disabled="girando || apuesta < 1" @click="retirar">
              Retirar
            </button>
            <button
              v-if="ultimaApuesta > 0"
              class="btn"
              :disabled="girando || apuesta > 0 || ultimaApuesta > banca.dinero"
              @click="repetir"
            >
              Repetir {{ ultimaApuesta }}€
            </button>
          </div>
            <div class="bombillas" aria-hidden="true">
              <span v-for="n in 12" :key="n" class="bombilla"></span>
            </div>
        </div>

        <div class="palanca" :class="{ suelta: !arrastrando }" :style="{ '--y': palancaY + 'px' }">
          <div class="carril"></div>
          <div class="vastago"></div>
          <span class="base"></span>
          <button
            class="bola"
            type="button"
            :disabled="girando"
            aria-label="Palanca: arrástrala hacia abajo o púlsala para jugar"
            @pointerdown="alPresionar"
            @pointermove="alMover"
            @pointerup="alSoltar"
            @pointercancel="alCancelar"
            @keydown.enter.prevent="tirarConClick"
            @keydown.space.prevent="tirarConClick"
          ></button>
        </div>
      </div>

      <p class="ayuda">Arrastra la palanca hacia abajo (o púlsala) para jugar</p>

      <details class="premios">
        <summary>Tabla de premios</summary>
        <ul>
          <li v-for="p in tablaPremios" :key="p.icono">
            <span>{{ p.icono }}{{ p.icono }}{{ p.icono }}</span>
            <strong>×{{ p.premio }}</strong>
          </li>
          <li>
            <span>{{ CEREZA }}{{ CEREZA }}</span>
            <strong>×{{ PREMIO_DOS_CEREZAS }}</strong>
          </li>
        </ul>
      </details>
    </div>
  </section>
</template>

<style scoped src="../assets/tragaperras.css"></style>