<template>
  <div id="app">
    <div style="text-align: center;left: 45%; position: absolute; z-index: 9999;">
      <button @click="goToDemo(1)">demo 1</button>
      &nbsp;
      <!-- <button @click="goToDemo(2)">demo 2</button> -->
      &nbsp;
      <button @click="goToDemo(3)">demo 3</button>
    </div>
    <demo-one v-if="demo == '1'"></demo-one>
    <demo-two v-else-if="demo == '2'"></demo-two>
    <demo-three v-else-if="demo == '3'"></demo-three>
  </div>
</template>

<script>
import DemoOne from './components/DemoOne.vue';
import DemoTwo from './components/DemoTwo.vue';
import DemoThree from './components/DemoThree.vue';

export default {
  name: 'App',
   components: {
         'demo-one': DemoOne,
         'demo-two': DemoTwo,
         'demo-three': DemoThree
       },
  data() {
    return {
     demo:'1',
     showDemo1: true,
     showDemo2: false,
     showDemo3: false,
    }
  },
  watch(){

  },
  destroyed() {
  },
  mounted() {
window.addEventListener('popstate', this.syncFromUrl);
  },
  created(){
this.syncFromUrl();
  },
  beforeDestroy() {
    window.removeEventListener('popstate', this.syncFromUrl);
  },
  methods: {
    syncFromUrl() {
      const urlParams = new URLSearchParams(window.location.search);
      this.demo = urlParams.get('demo');
    },
    goToDemo(number) {
      const newUrl = `/?demo=${number}`;
      window.history.pushState({}, '', newUrl); 
      this.syncFromUrl(); 
    }
  }
}
</script>

<style>
body { 
  margin: 0; 
  overflow: hidden; 
  touch-action: none;
}
#app { 
  width: 100vw; 
  height: 100vh; 
}
</style>