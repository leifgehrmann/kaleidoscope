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

const autoRotationBarPercentage = ref(0);
const isUserAutoRotating = ref(false);
const selectedIndex = ref(0);
function updateSelectedIndex (value: number) {
  selectedIndex.value = value;
}

function updateAutoRotationBarPercentage (value: number) {
  autoRotationBarPercentage.value = Math.min(-1 / (Math.abs(value * 20) + 1) + 1, 1) * Math.sign(value);
}
function updateIsUserAutoRotating(value: boolean) {
  isUserAutoRotating.value = value;
}
</script>

<template>
  <Kaleidoscope
    :scope-shape="selectedIndex"
    @update:scope-rotation-velocity="updateAutoRotationBarPercentage"
    @update:is-user-auto-rotating="updateIsUserAutoRotating"
  />
  <div
    class="absolute bottom-0 w-full flex flex-col justify-center gap-1 px-2 py-1 items-center"
    style="bottom: calc(env(safe-area-inset-bottom))"
  >
    <div
      class="
        w-full
        max-w-[10.5rem]
        relative
        pb-1
      "
    >
      <div
        class="
        p-0.5
        bg-neutral-800
        bg-opacity-70
        backdrop-blur-xl
        rounded-xl
        absolute
        transition-opacity ease-in
      "
        :style="{
          width: 'calc(0.25rem + (' + (Math.abs(autoRotationBarPercentage) * 50) + '%))',
          left: 'calc(' + ((autoRotationBarPercentage > 0 ? 0.5 : (0.5 - Math.abs(autoRotationBarPercentage) * 0.5)) * 100) + '% - 0.25rem / 2)',
          opacity: (isUserAutoRotating && Math.abs(autoRotationBarPercentage) > 0.01) ? '100%' : '0%',
        }"
      />
    </div>
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
