<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import SegmentedControl, { Option } from './components/SegmentedControl.vue';
import { ref } from 'vue';

const options: Option[] = [
  {
    buttonAriaLabel: 'Equilateral',
    imageUrl: '/equilateral.svg',
  },
  {
    buttonAriaLabel: 'Square',
    imageUrl: '/square.svg',
  },
  {
    buttonAriaLabel: 'Isosceles',
    imageUrl: '/isosceles.svg',
  },
  {
    buttonAriaLabel: 'Scalene',
    imageUrl: '/scalene.svg',
  }
];
const selectedIndex = ref(0);
function updateSelectedIndex (value: number) {
  selectedIndex.value = value;
}
</script>

<template>
  <Kaleidoscope :scope-shape="selectedIndex" />
  <div
    class="absolute bottom-0 w-full flex justify-center gap-2 px-2 py-1 items-center"
    style="bottom: calc(env(safe-area-inset-bottom))"
  >
    <SegmentedControl
      class="w-full max-w-48"
      :options="options"
      :selected-index="selectedIndex"
      @update:selected-index="updateSelectedIndex"
    />
  </div>
</template>

<style scoped>
</style>
