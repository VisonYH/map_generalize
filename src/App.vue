<template>
  <div id="app">
    <v-header></v-header>
    <v-map></v-map>
    <router-view/>
  </div>
</template>

<script>
import vHeader from './components/header/vHeader.vue'
import vMap from './components/map/map'
export default {
  name: 'App',
  data () {
    return {
    }
  },
  components: {
    vHeader,
    vMap
  }
}
</script>

<style lang="less">
html, body{
  height: 100%;
}
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  height: 100%;
  padding-top: 40px;
}
</style>
