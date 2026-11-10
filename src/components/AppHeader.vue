<script setup lang="ts">
import { ref } from 'vue'
import { useDineroStore } from '../stores/dinero'

const banca = useDineroStore()

const menuAbierto = ref(false)

function cerrarMenu() {
  menuAbierto.value = false
}
</script>

<template>
  <header class="app-header" @keydown.esc="cerrarMenu">
    <RouterLink to="/" class="marca-link" @click="cerrarMenu">
      <div class="marca">
        <span class="logo" aria-hidden="true">♠</span>
        <span class="titulo">Litro Vegas</span>
      </div>
    </RouterLink>

    <nav
      id="menu-juegos"
      class="nav"
      :class="{ abierto: menuAbierto }"
      @click="cerrarMenu"
    >
      <slot name="nav" />
    </nav>

    <div class="cartera">
      <button
        v-if="banca.dinero < 1"
        class="reinicio-dinero"
        @click="banca.reiniciar()"
      >
        Reiniciar dinero
      </button>

      <div class="dinero">
        <span class="dinero-etiqueta">Saldo</span>
        <span class="dinero-valor">{{ banca.dinero }}€</span>
      </div>
    </div>

    <button
      class="burger"
      :class="{ abierto: menuAbierto }"
      type="button"
      :aria-expanded="menuAbierto"
      aria-controls="menu-juegos"
      :aria-label="menuAbierto ? 'Cerrar menú de juegos' : 'Abrir menú de juegos'"
      @click="menuAbierto = !menuAbierto"
    >
      <span></span>
      <span></span>
      <span></span>
    </button>

    <div v-if="menuAbierto" class="velo" @click="cerrarMenu"></div>
  </header>
</template>

<style scoped src="../assets/appHeader.css"></style>