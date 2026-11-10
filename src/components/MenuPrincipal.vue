<script setup lang="ts">
import { useRouter } from 'vue-router'
import { JUEGOS, type Juego } from '../data/juegos'

const router = useRouter()

function elegir(juego: Juego) {
  if (juego.disponible) router.push(`/${juego.id}`)
}

// Abanico: las cartas se inclinan según su posición respecto al centro
function giro(indice: number) {
  const centro = (JUEGOS.length - 1) / 2
  return `${(indice - centro) * 5}deg`
}
</script>

<template>
  <section class="menu">
    <header class="cabecera">
      <h1>Litro Vegas</h1>
      <p>Elige tu juego</p>
    </header>

    <ul class="juegos">
      <li v-for="(juego, i) in JUEGOS" :key="juego.id">
        <button
          class="carta-juego"
          :class="{ bloqueado: !juego.disponible }"
          :disabled="!juego.disponible"
          :style="{ '--acento': juego.color, '--giro': giro(i), '--i': i }"
          :aria-label="juego.disponible ? `Jugar a ${juego.nombre}` : `${juego.nombre}, próximamente`"
          @click="elegir(juego)"
        >
          <span class="esquina esquina-sup" aria-hidden="true">{{ juego.simbolo }}</span>

          <span class="icono" aria-hidden="true">{{ juego.icono }}</span>
          <span class="nombre">{{ juego.nombre }}</span>
          <span class="descripcion">{{ juego.descripcion }}</span>
          <span class="accion">{{ juego.disponible ? 'Jugar' : 'Próximamente' }}</span>

          <span class="esquina esquina-inf" aria-hidden="true">{{ juego.simbolo }}</span>
        </button>
      </li>
    </ul>
  </section>
</template>

<style scoped src="../assets/menuPrincipal.css"></style>