<template>
  <div id="app">
    <Tokens />
  </div>
</template>

<script>
import Tokens from './components/Tokens.vue';

export default {
  name: 'App',
  components: {
    Tokens,
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
