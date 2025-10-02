<script setup lang="ts">
import { ref } from 'vue'

const counter = ref(110)

const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>SALUT LE JIIIIIIISTRERIIIIEWWWW</h1>
  <p>Counter {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
