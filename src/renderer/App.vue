<template>
  <div id="app" style="height: 100%">
    <router-view></router-view>
  </div>
</template>

<script>
  export default {
    name: 'my-book'
  }
</script>

<style>
  /* CSS */
  html, body {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
  }

  button, a, input, textarea {
    -webkit-app-region: no-drag;
  }

  .noDrag {
    -webkit-app-region: no-drag;
  }
</style>
