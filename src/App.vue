<template>
  <div id="app">
    <img-upload v-on:select-complete="secelted"></img-upload> 

    
  </div>
</template>

<script>
import upload from "./components/upload-img.vue";
export default {
  name: 'app',
  data () {
    return {
      
    }
  },
  components:{
    'img-upload': upload
  },
  methods:{
    //选择图片后返回的数据
    secelted(data){
      console.log(data)
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

h1, h2 {
  font-weight: normal;
}
body{
  background-color: #000;
}
ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
</style>
