<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import SegmentedControl, { Option } from './components/SegmentedControl.vue';
import { ref } from 'vue';
import SteeringControl from './components/SteeringControl.vue';

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
const scopeRotationVelocity = ref(0);
function updateSelectedIndex (value: number) {
  selectedIndex.value = value;
}

function updateScopeRotationVelocity(value: number) {
  scopeRotationVelocity.value = value;
}

</script>

<template>
  <Kaleidoscope
    :scope-shape="selectedIndex"
    :scope-rotation-velocity="scopeRotationVelocity"
  />
  <div
    class="absolute bottom-0 w-full flex flex-col justify-center gap-1 px-2 py-1 items-center pointer-events-none"
    style="bottom: calc(env(safe-area-inset-bottom))"
  >
    <SteeringControl
      class="w-full max-w-48 h-6"
      @update:velocity="updateScopeRotationVelocity"
    />
    <SegmentedControl
      class="w-full max-w-48 pointer-events-auto"
      :options="options"
      :selected-index="selectedIndex"
      @update:selected-index="updateSelectedIndex"
    />
  </div>
</template>

<style scoped>
</style>
