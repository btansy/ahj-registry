<template>
  <div id="app">
    <Navbar id="navbar"/>
    <router-view id="router-v" />
  </div>
</template>

<script>
import Navbar from "./components/Navbar.vue";
export default {
  components: {
    Navbar
  }
};
</script>

<style>
#app {
  font-family: "Roboto Condensed";

  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  background-color: white;
  display: grid;
  grid-template-rows: 4.25em auto;
  height: 100vh;
}

#router-v {
}

#navbar {
  grid-row: 1 / 2;
}
</style>
