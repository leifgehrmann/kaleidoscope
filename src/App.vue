<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import SegmentedControl from './components/SegmentedControl.vue';
import {ref} from 'vue';

const options = [
    '▲', '◣', '◣', '⏹'
];
const selectedIndex = ref(0);
function updateSelectedIndex (value: number) {
  selectedIndex.value = value;
}
</script>

<template>
  <Kaleidoscope :scope-shape="selectedIndex" />
  <div class="absolute bottom-0 w-full flex justify-center">
    <SegmentedControl
      class="w-full max-w-48 m-2"
      :options="options"
      :selected-index="selectedIndex"
      @update:selected-index="updateSelectedIndex"
    />
  </div>
</template>

<style scoped>
</style>
