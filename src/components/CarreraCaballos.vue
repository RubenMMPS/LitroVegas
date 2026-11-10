
<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import type { Caballo } from '../models/Horse'
import { useApuesta } from '../composables/useApuesta'
import ApuestaForm from './ApuestaForm.vue'

const { apuesta, errorApuesta, apostar, ganar, reiniciarApuesta } = useApuesta()

const carreraEnCurso = ref(false)
const caballoElegido = ref<number | null>(null)
const ganadorCarrera = ref<Caballo | null>(null)

let temporizador: ReturnType<typeof setInterval> | undefined

const caballos = ref<Caballo[]>([
  {
    id: 1, name: "Secretariat", color: 'rojo', posicionX: 0, img: 'caballoRojo.png',
    cuota: 1.8, avanceMin: 3, avanceMax: 10
  },
  {
    id: 2, name: "Oguri Cap", color: 'azul', posicionX: 0, img: 'caballoAzul.png',
    cuota: 2.5, avanceMin: 2, avanceMax: 10
  },
  {
    id: 3, name: "Deep Impact", color: 'verde', posicionX: 0, img: 'caballoVerde.png',
    cuota: 4, avanceMin: 1, avanceMax: 10
  },
  {
    id: 4, name: "Vodka", color: 'amarillo', posicionX: 0, img: 'caballoAmarillo.png',
    cuota: 6, avanceMin: 0, avanceMax: 10
  }
])

function avanzar(caballo: Caballo) {
  const avance = Math.floor(
    Math.random() * (caballo.avanceMax - caballo.avanceMin + 1)
  ) + caballo.avanceMin

  caballo.posicionX += avance
}

function iniciar_carrera() {
  if (carreraEnCurso.value || caballoElegido.value === null) return

  caballos.value.forEach(caballo => {
    caballo.posicionX = 0
  })

  ganadorCarrera.value = null
  carreraEnCurso.value = true

  temporizador = setInterval(() => {
    caballos.value.forEach(caballo => {
      if (caballo.posicionX < 100) {
        avanzar(caballo)
      }
    })

    const finalistas = caballos.value.filter(
      caballo => caballo.posicionX >= 100
    )

    if (finalistas.length > 0) {
      ganadorCarrera.value = finalistas.reduce((anterior, actual) =>
        actual.posicionX > anterior.posicionX ? actual : anterior
      )

      if (temporizador !== undefined) {
        clearInterval(temporizador)
        temporizador = undefined
      }
      calcularResultado();
      carreraEnCurso.value = false
    }
  }, 500)
}

function reiniciarCarrera() {
  if (temporizador !== undefined) {
    clearInterval(temporizador)
    temporizador = undefined
  }

  caballos.value.forEach(caballo => {
    caballo.posicionX = 0
  })

  caballoElegido.value = null
  ganadorCarrera.value = null
  carreraEnCurso.value = false
  reiniciarApuesta()
}

onUnmounted(() => {
  if (temporizador !== undefined) {
    clearInterval(temporizador)
  }
})

function calcularResultado(){
    if(ganadorCarrera.value?.id === caballoElegido.value){
        ganar(ganadorCarrera.value.cuota);
    }
}
</script>

<template>
  <div class="carrera-caballos">
    <div v-if="apuesta < 1">
    <section class="seleccion-caballo">
        <h2>¿Quién ganará la carrera?</h2>

        <label class="caballo-label" v-for="caballo in caballos" :key="caballo.id">
          <input
            type="radio"
            :value="caballo.id"
            v-model="caballoElegido"
            :disabled="carreraEnCurso"
          />

          <span class="caballo-nombre">{{ caballo.name }}</span>
          <span class="caballo-cuota">x{{ caballo.cuota.toFixed(2) }}</span>
        </label>
      </section>
      <ApuestaForm
      v-if="caballoElegido"
      :error="errorApuesta"
      @apostar="apostar"
        />
    </div>
    

    <template v-else>
      <h1 class="titulo" :class="{ 'titulo--minimizado': ganadorCarrera }">Carrera de Caballos</h1>

      <div class="pista">
        <div class="meta"></div>

        <div
          v-for="caballo in caballos"
          :key="caballo.id"
          class="carril"
        >
          <span class="carril__numero">{{ caballo.id }}</span>

          <div
            class="caballo"
            :style="{ '--pos': Math.min(caballo.posicionX, 100) }"
          >
            <img
              :src="`/caballos/${caballo.img}`"
              :alt="`Caballo ${caballo.color}`"
            />
            <span v-if="caballo.id === caballoElegido" class="mi-caballo">Tú</span>
          </div>
        </div>
      </div>

      <button
      v-if="!ganadorCarrera"
        class="boton"
        @click="iniciar_carrera"
        :disabled="carreraEnCurso || caballoElegido === null"
      >
        {{ carreraEnCurso ? 'Carrera en curso...' : 'Iniciar carrera' }}
      </button>

      <div v-if="ganadorCarrera" class="resultado"">
        <h2>¡Ha ganado {{ ganadorCarrera.name }}, el caballo {{ ganadorCarrera.color }}!</h2>

        <p v-if="ganadorCarrera.id === caballoElegido">
          ¡Has acertado! Cuota: {{ ganadorCarrera.cuota.toFixed(2) }}
        </p>

        <p v-else>
          Esta vez no has acertado. ¡Suerte en la próxima carrera!
        </p>

        <button class="boton" @click="reiniciarCarrera">
          Nueva carrera
        </button>
      </div>
    </template>
  </div>
</template>

<style scoped src="../assets/carreraCaballos.css"></style>