<script setup>
import { ref } from 'vue'
const count = ref(0)
</script>

<template>
  <p>Componente actual: A</p>
  <span style="margin-right: 20px">Contador: {{ count }}</span>
  <button @click="count++">+</button>
</template>
