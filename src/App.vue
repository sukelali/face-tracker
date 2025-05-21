<template>
  <div id="app">
    <FaceTracker />
    <!-- <ModelLoader /> -->
  </div>
</template>

<script setup>
import FaceTrackerMediaPipe from './components/FaceTrackerMediaPipe.vue';
import ModelLoader from './components/ModelLoader.vue';
import FaceTracker from './components/FaceTracker.vue';
</script>

<style>
#app {
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>