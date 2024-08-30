<template>
  <div id="app">
    <FaceDetection />
  </div>
</template>

<script>
import FaceDetection from './components/FaceDetection.vue';

export default {
  name: 'App',
  components: {
    FaceDetection,
  },
};
</script>

<style>
/* Add any global styles here */
</style>
