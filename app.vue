<template>
  <main>
    <header>
      <h1>Agua</h1>
    </header>
    <Wave />
  </main>
</template>

<script lang="ts">
import Wave from "./components/Wave.vue";
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Barrio&family=Poppins&display=swap');

html,
body {
  background-color: #f5f5f5;
}

html,
body {
  margin: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

main {
  z-index: 1;
}

header h1 {
  font-family: 'Barrio', cursive;
  text-align: center;
  color: #EAA634;
  font-size: 4.5rem;
  position: relative;
  z-index: 1;
  text-shadow:
    -2px -2px 0 #000,
    2px -2px 0 #000,
    -2px 2px 0 #000,
    2px 2px 0 #000;

}
</style>
