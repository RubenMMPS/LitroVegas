<script setup lang="ts">
import { computed, onBeforeUnmount, ref } from 'vue'
import { useDineroStore } from '../stores/dinero'
import { TIPOS_FICHA, desglosarApuesta } from '../utils/fichas'
import FichasApuesta from './FichasApuesta.vue'

const ORDEN = [
  0, 32, 15, 19, 4, 21, 2, 25, 17, 34, 6, 27, 13, 36, 11, 30, 8, 23, 10, 5, 24, 16, 33, 1, 20, 14,
  31, 9, 22, 18, 29, 7, 28, 12, 35, 3, 26
]
const SLOT = 360 / ORDEN.length
const ROJOS = new Set([1, 3, 5, 7, 9, 12, 14, 16, 18, 19, 21, 23, 25, 27, 30, 32, 34, 36])

type Color = 'verde' | 'rojo' | 'negro'
const COLORES_HEX: Record<Color, string> = { verde: '#1f7a3a', rojo: '#c0281c', negro: '#151515' }

function colorDe(n: number): Color {
  if (n === 0) return 'verde'
  return ROJOS.has(n) ? 'rojo' : 'negro'
}

const FONDO_RUEDA = `conic-gradient(from ${-SLOT / 2}deg, ${ORDEN.map(
  (n, k) =>
    `${COLORES_HEX[colorDe(n)]} ${(k * SLOT).toFixed(3)}deg ${((k + 1) * SLOT).toFixed(3)}deg`
).join(', ')})`

interface Definicion {
  multiplicador: number
  gana: (n: number) => boolean
}

const DEFINICIONES: Record<string, Definicion> = {}

for (let n = 0; n <= 36; n++) {
  DEFINICIONES[`n-${n}`] = { multiplicador: 36, gana: (x) => x === n }
}
Object.assign(DEFINICIONES, {
  rojo: { multiplicador: 2, gana: (x: number) => ROJOS.has(x) },
  negro: { multiplicador: 2, gana: (x: number) => x > 0 && !ROJOS.has(x) },
  par: { multiplicador: 2, gana: (x: number) => x > 0 && x % 2 === 0 },
  impar: { multiplicador: 2, gana: (x: number) => x % 2 === 1 },
  falta: { multiplicador: 2, gana: (x: number) => x >= 1 && x <= 18 },
  pasa: { multiplicador: 2, gana: (x: number) => x >= 19 },
  doc1: { multiplicador: 3, gana: (x: number) => x >= 1 && x <= 12 },
  doc2: { multiplicador: 3, gana: (x: number) => x >= 13 && x <= 24 },
  doc3: { multiplicador: 3, gana: (x: number) => x >= 25 && x <= 36 },
  col1: { multiplicador: 3, gana: (x: number) => x > 0 && x % 3 === 1 },
  col2: { multiplicador: 3, gana: (x: number) => x > 0 && x % 3 === 2 },
  col3: { multiplicador: 3, gana: (x: number) => x > 0 && x % 3 === 0 }
})

interface Componente {
  numeros: number[]
  multiplicador: number
  fichas: number
}

interface Anunciada {
  clave: string
  nombre: string
  unidades: number
  componentes: Componente[]
}

const pleno = (n: number): Componente => ({ numeros: [n], multiplicador: 36, fichas: 1 })
const caballo = (a: number, b: number): Componente => ({
  numeros: [a, b],
  multiplicador: 18,
  fichas: 1
})

const ANUNCIADAS: Record<string, Anunciada> = {}

function registrar(clave: string, nombre: string, componentes: Componente[]) {
  ANUNCIADAS[clave] = {
    clave,
    nombre,
    componentes,
    unidades: componentes.reduce((total, c) => total + c.fichas, 0)
  }
}

registrar('a-huerfanos', 'Huérfanos', [
  pleno(1),
  caballo(6, 9),
  caballo(14, 17),
  caballo(17, 20),
  caballo(31, 34)
])

registrar('a-vecinos', 'Vecinos del cero', [
  { numeros: [0, 2, 3], multiplicador: 12, fichas: 2 },
  caballo(4, 7),
  caballo(12, 15),
  caballo(18, 21),
  caballo(19, 22),
  caballo(32, 35),
  { numeros: [25, 26, 28, 29], multiplicador: 9, fichas: 2 }
])

registrar('a-tercio', 'Tercio del cilindro', [
  caballo(5, 8),
  caballo(10, 11),
  caballo(13, 16),
  caballo(23, 24),
  caballo(27, 30),
  caballo(33, 36)
])

registrar('a-cero', 'Juego del cero', [caballo(0, 3), caballo(12, 15), pleno(26), caballo(32, 35)])

for (let n = 0; n <= 36; n++) {
  const k = ORDEN.indexOf(n)
  registrar(
    `v-${n}`,
    `Vecinos del ${n}`,
    [-2, -1, 0, 1, 2].map((d) => pleno(ORDEN[(k + d + ORDEN.length) % ORDEN.length]!))
  )
}

const LISTA_ANUNCIADAS = ['a-huerfanos', 'a-vecinos', 'a-tercio', 'a-cero'].map(
  (clave) => ANUNCIADAS[clave]!
)

function numerosCubiertos(clave: string): number[] {
  const anunciada = ANUNCIADAS[clave]
  if (!anunciada) return []
  return [...new Set(anunciada.componentes.flatMap((c) => c.numeros))]
}

function descripcion(anunciada: Anunciada) {
  return `${anunciada.nombre}: ${anunciada.unidades} fichas, cubre ${numerosCubiertos(anunciada.clave).length} números`
}

function pago(clave: string, importe: number, numero: number): number {
  const simple = DEFINICIONES[clave]
  if (simple) return simple.gana(numero) ? importe * simple.multiplicador : 0

  const anunciada = ANUNCIADAS[clave]
  if (!anunciada) return 0

  const unidad = importe / anunciada.unidades
  return anunciada.componentes.reduce(
    (total, c) =>
      c.numeros.includes(numero) ? total + unidad * c.fichas * c.multiplicador : total,
    0
  )
}

interface Celda {
  clave: string
  etiqueta: string
  n?: number
  c: number
  cs: number
  f: number
  fs: number
  clase: string
}

const CELDAS: Celda[] = []

CELDAS.push({ clave: 'n-0', etiqueta: '0', n: 0, c: 1, cs: 1, f: 1, fs: 3, clase: 'numero verde' })

for (let r = 0; r < 3; r++) {
  for (let c = 0; c < 12; c++) {
    const n = c * 3 + (3 - r)
    CELDAS.push({
      clave: `n-${n}`,
      etiqueta: String(n),
      n,
      c: c + 2,
      cs: 1,
      f: r + 1,
      fs: 1,
      clase: `numero ${colorDe(n)}`
    })
  }
}
;(['col3', 'col2', 'col1'] as const).forEach((clave, r) => {
  CELDAS.push({ clave, etiqueta: '2:1', c: 14, cs: 1, f: r + 1, fs: 1, clase: 'exterior columna' })
})
;(
  [
    ['doc1', '1ª docena'],
    ['doc2', '2ª docena'],
    ['doc3', '3ª docena']
  ] as const
).forEach(([clave, etiqueta], i) => {
  CELDAS.push({ clave, etiqueta, c: 2 + i * 4, cs: 4, f: 4, fs: 1, clase: 'exterior' })
})
;(
  [
    ['falta', '1-18', ''],
    ['par', 'Par', ''],
    ['rojo', 'Rojo', 'rojo'],
    ['negro', 'Negro', 'negro'],
    ['impar', 'Impar', ''],
    ['pasa', '19-36', '']
  ] as const
).forEach(([clave, etiqueta, color], i) => {
  CELDAS.push({
    clave,
    etiqueta,
    c: 2 + i * 2,
    cs: 2,
    f: 5,
    fs: 1,
    clase: `exterior ${color}`
  })
})

const FICHAS_APUESTA = [...TIPOS_FICHA].reverse().filter((t) => t.valor <= 100)

const banca = useDineroStore()

const apuestas = ref<Record<string, number>>({})
const historial = ref<{ clave: string; valor: number }[]>([])
const apuestasAnteriores = ref<Record<string, number>>({})
const fichaActiva = ref(5)
const vecinoDe = ref(17)
const resaltados = ref<number[]>([])

const girando = ref(false)
const resultado = ref<number | null>(null)
const premio = ref(0)
const ultimos = ref<number[]>([])
const mensaje = ref('Elige una ficha y haz tus apuestas en el tablero.')

const ruedaGiro = ref(0)
const bolaGiro = ref(0)

const totalApostado = computed(() => Object.values(apuestas.value).reduce((a, b) => a + b, 0))
const totalAnterior = computed(() =>
  Object.values(apuestasAnteriores.value).reduce((a, b) => a + b, 0)
)

const numeroVecino = computed(() => {
  const v = Math.floor(Number(vecinoDe.value))
  return Number.isFinite(v) ? Math.min(36, Math.max(0, v)) : 0
})
const anunciadaVecinos = computed(() => ANUNCIADAS[`v-${numeroVecino.value}`]!)
const textoVecinos = computed(() =>
  numerosCubiertos(anunciadaVecinos.value.clave).join(' · ')
)

const esperar = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms))

onBeforeUnmount(() => {
  if (totalApostado.value > 0 && !girando.value) banca.sumar(totalApostado.value)
})

function apostar(clave: string) {
  if (girando.value) return
  if (!banca.restar(fichaActiva.value)) {
    mensaje.value = 'No tienes saldo suficiente para esa ficha.'
    return
  }
  apuestas.value[clave] = (apuestas.value[clave] ?? 0) + fichaActiva.value
  historial.value.push({ clave, valor: fichaActiva.value })
  mensaje.value = 'Cuando estés listo, pulsa «Girar».'
}

function apostarAnunciada(clave: string) {
  if (girando.value) return
  const anunciada = ANUNCIADAS[clave]
  if (!anunciada) return
  const coste = anunciada.unidades * fichaActiva.value
  if (!banca.restar(coste)) {
    mensaje.value = 'No tienes saldo suficiente para esa apuesta.'
    return
  }
  apuestas.value[clave] = (apuestas.value[clave] ?? 0) + coste
  historial.value.push({ clave, valor: coste })
  mensaje.value = `${anunciada.nombre}: ${coste}€ repartidos en ${anunciada.unidades} fichas.`
}

function resaltar(clave: string) {
  resaltados.value = numerosCubiertos(clave)
}

function quitarResalte() {
  resaltados.value = []
}

function estaResaltada(celda: Celda) {
  return celda.n !== undefined && resaltados.value.includes(celda.n)
}

function deshacer() {
  if (girando.value) return
  const ultima = historial.value.pop()
  if (!ultima) return
  const restante = (apuestas.value[ultima.clave] ?? 0) - ultima.valor
  if (restante > 0) apuestas.value[ultima.clave] = restante
  else delete apuestas.value[ultima.clave]
  banca.sumar(ultima.valor)
}

function limpiar() {
  if (girando.value || totalApostado.value < 1) return
  banca.sumar(totalApostado.value)
  apuestas.value = {}
  historial.value = []
  mensaje.value = 'Mesa limpia.'
}

function repetir() {
  if (girando.value || totalApostado.value > 0 || totalAnterior.value < 1) return
  if (!banca.restar(totalAnterior.value)) {
    mensaje.value = 'No tienes saldo suficiente para repetir.'
    return
  }
  apuestas.value = { ...apuestasAnteriores.value }
  historial.value = Object.entries(apuestasAnteriores.value).map(([clave, valor]) => ({
    clave,
    valor
  }))
  mensaje.value = 'Apuestas repetidas. Pulsa «Girar».'
}

function estiloMarca(clave: string) {
  const importe = apuestas.value[clave]
  if (!importe) return {}
  const tipo = desglosarApuesta(importe)[0]!.tipo
  return { '--color': tipo.color, '--borde': tipo.borde, '--texto': tipo.texto }
}

function etiquetaImporte(importe: number) {
  return importe >= 1000 ? `${importe / 1000}K` : String(importe)
}

async function girar() {
  if (girando.value || totalApostado.value < 1) return

  girando.value = true
  resultado.value = null
  premio.value = 0
  resaltados.value = []
  mensaje.value = '¡No va más!'

  const apostado = { ...apuestas.value }
  const total = totalApostado.value
  apuestasAnteriores.value = apostado

  const numero = ORDEN[Math.floor(Math.random() * ORDEN.length)]!
  const k = ORDEN.indexOf(numero)

  const objetivo = -k * SLOT
  const resto = (((objetivo - ruedaGiro.value) % 360) + 360) % 360
  ruedaGiro.value += 5 * 360 + resto

  const m = ((bolaGiro.value % 360) + 360) % 360
  bolaGiro.value = bolaGiro.value - 7 * 360 - m

  await esperar(6200)

  let devuelto = 0
  for (const [clave, importe] of Object.entries(apostado)) {
    devuelto += pago(clave, importe, numero)
  }
  if (devuelto > 0) banca.sumar(devuelto)

  resultado.value = numero
  premio.value = devuelto
  ultimos.value = [numero, ...ultimos.value].slice(0, 10)

  const nombreColor = colorDe(numero)
  if (devuelto > 0) {
    const neto = devuelto - total
    mensaje.value = `Sale el ${numero} ${nombreColor}. Cobras ${devuelto}€ (${neto >= 0 ? '+' : ''}${neto}€).`
  } else {
    mensaje.value = `Sale el ${numero} ${nombreColor}. Pierdes ${total}€.`
  }

  apuestas.value = {}
  historial.value = []
  girando.value = false
}
</script>

<template>
  <section class="ruleta">
    <div class="contenido">
      <div class="zona">
        <div class="rueda-zona">
          <div class="puntero" aria-hidden="true"></div>

          <div
            class="rueda-wrap"
            :class="{ girando }"
            :style="{ '--giro': ruedaGiro + 'deg', '--giro-bola': bolaGiro + 'deg', '--slot': SLOT + 'deg' }"
          >
            <div class="rueda" :style="{ background: FONDO_RUEDA }">
              <span
                v-for="(n, k) in ORDEN"
                :key="n"
                class="num"
                :style="{ transform: `rotate(${k * SLOT}deg)` }"
              >
                <b>{{ n }}</b>
              </span>
            </div>

            <div class="centro" :class="resultado === null ? '' : colorDe(resultado)">
              <span>{{ resultado ?? '♠' }}</span>
            </div>

            <div class="orbita" aria-hidden="true">
              <span class="bola-r" :class="{ cae: !girando }"></span>
            </div>
          </div>

          <ul class="ultimos" aria-label="Últimos números">
            <li v-for="(n, i) in ultimos" :key="i" :class="colorDe(n)">{{ n }}</li>
          </ul>
        </div>

        <div class="tablero">
          <div class="mesa-apuestas">
            <button
              v-for="c in CELDAS"
              :key="c.clave"
              class="celda"
              :class="[c.clase, { resaltada: estaResaltada(c) }]"
              :style="{ '--c': c.c, '--cs': c.cs, '--f': c.f, '--fs': c.fs }"
              :disabled="girando"
              :aria-label="`Apostar a ${c.etiqueta}`"
              @click="apostar(c.clave)"
            >
              {{ c.etiqueta }}
              <span v-if="apuestas[c.clave]" class="marca" :style="estiloMarca(c.clave)">
                {{ etiquetaImporte(apuestas[c.clave]!) }}
              </span>
            </button>
          </div>
        </div>
      </div>

      <div class="anunciadas" role="group" aria-label="Apuestas anunciadas">
        <span class="anunciadas-titulo">Apuestas anunciadas</span>

        <button
          v-for="a in LISTA_ANUNCIADAS"
          :key="a.clave"
          class="anunciada"
          :disabled="girando"
          :title="descripcion(a)"
          @click="apostarAnunciada(a.clave)"
          @mouseenter="resaltar(a.clave)"
          @mouseleave="quitarResalte"
          @focus="resaltar(a.clave)"
          @blur="quitarResalte"
        >
          {{ a.nombre }}
          <small>{{ a.unidades }} × {{ fichaActiva }}€ = {{ a.unidades * fichaActiva }}€</small>
          <span v-if="apuestas[a.clave]" class="marca" :style="estiloMarca(a.clave)">
            {{ etiquetaImporte(apuestas[a.clave]!) }}
          </span>
        </button>

        <div class="vecinos">
          <label for="vecinos-numero">Vecinos del</label>
          <input
            id="vecinos-numero"
            v-model.number="vecinoDe"
            type="number"
            min="0"
            max="36"
            :disabled="girando"
          />
          <button
            class="anunciada"
            :disabled="girando"
            :title="`Cubre: ${textoVecinos}`"
            @click="apostarAnunciada(anunciadaVecinos.clave)"
            @mouseenter="resaltar(anunciadaVecinos.clave)"
            @mouseleave="quitarResalte"
            @focus="resaltar(anunciadaVecinos.clave)"
            @blur="quitarResalte"
          >
            Apostar
            <small>
              {{ anunciadaVecinos.unidades }} × {{ fichaActiva }}€ =
              {{ anunciadaVecinos.unidades * fichaActiva }}€
            </small>
            <span
              v-if="apuestas[anunciadaVecinos.clave]"
              class="marca"
              :style="estiloMarca(anunciadaVecinos.clave)"
            >
              {{ etiquetaImporte(apuestas[anunciadaVecinos.clave]!) }}
            </span>
          </button>
          <span class="vecinos-numeros">{{ textoVecinos }}</span>
        </div>
      </div>

      <p class="mensaje" aria-live="polite">{{ mensaje }}</p>

      <div class="controles">
        <div class="fichas-selector" role="group" aria-label="Elegir ficha">
          <button
            v-for="t in FICHAS_APUESTA"
            :key="t.valor"
            class="ficha-boton"
            :class="{ activa: fichaActiva === t.valor }"
            :style="{ '--color': t.color, '--borde': t.borde, '--texto': t.texto }"
            :aria-pressed="fichaActiva === t.valor"
            :aria-label="`Ficha de ${t.valor} euros`"
            @click="fichaActiva = t.valor"
          >
            {{ t.valor }}
          </button>
        </div>

        <div class="total">
          <FichasApuesta :cantidad="totalApostado" />
          <span>Apuesta total: <strong>{{ totalApostado }}€</strong></span>
        </div>

        <div class="acciones">
          <button class="btn btn-secundario" :disabled="girando || !historial.length" @click="deshacer">
            Deshacer
          </button>
          <button class="btn btn-secundario" :disabled="girando || totalApostado < 1" @click="limpiar">
            Limpiar
          </button>
          <button
            v-if="totalAnterior > 0"
            class="btn"
            :disabled="girando || totalApostado > 0 || totalAnterior > banca.dinero"
            @click="repetir"
          >
            Repetir {{ totalAnterior }}€
          </button>
          <button class="btn btn-girar" :disabled="girando || totalApostado < 1" @click="girar">
            Girar
          </button>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped src="../assets/ruleta.css"></style>