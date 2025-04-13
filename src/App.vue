<script setup lang="ts">
import Calculator from './components/Calculator.vue'
</script>

<template>
  <hgroup>
    <h1>rekordbox Playlist Calculator</h1>
    <p>Calculate playing set times for your rekordbox playlists</p>
  </hgroup>
  <Calculator />
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
