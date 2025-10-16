<template>
  <div id="app">
    <CrewPlanning msg="Welcome to Your Vue.js App" />
  </div>
</template>

<script>
import CrewPlanning from "./components/CrewPlanning.vue";

export default {
  name: "App",
  components: {
    CrewPlanning,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
