<script setup lang="ts">
import BordCanvas from './components/BordCanvas.vue'
</script>

<template>
  <BordCanvas />
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family:
    -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  overflow: hidden;
}

#app {
  width: 100vw;
  height: 100vh;
  padding: 0;
  margin: 0;
}
</style>
