<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import SegmentedControl, { Option } from './components/SegmentedControl.vue';
import { ref } from 'vue';
import SteeringControl from './components/SteeringControl.vue';
import { ScopeShape } from './scopeShape.ts';
import Prompt from './components/Prompt.vue';

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

const showPrompt = ref(true);
const selectedIndex = ref(ScopeShape.Equilateral);
const scopeAutoRotationVelocity = ref(0);
function updateSelectedIndex (value: number) {
  selectedIndex.value = value;
}

function updateScopeAutoRotationVelocity(value: number) {
  scopeAutoRotationVelocity.value = value;
}

function dismissPrompt() {
  showPrompt.value = false;
}

</script>

<template>
  <Kaleidoscope
    v-if="!showPrompt"
    :scope-shape="selectedIndex"
    :scope-auto-rotation-velocity="scopeAutoRotationVelocity"
  />
  <div
    v-if="!showPrompt"
    class="absolute w-full flex flex-col justify-end gap-1 px-2 py-1 items-center pointer-events-none"
    style="bottom:calc(env(safe-area-inset-bottom))"
  >
    <SteeringControl
      class="w-full max-w-48 h-6"
      @update:velocity="updateScopeAutoRotationVelocity"
    />
    <SegmentedControl
      class="w-full max-w-48 pointer-events-auto"
      :options="options"
      :selected-index="selectedIndex"
      @update:selected-index="updateSelectedIndex"
    />
  </div>
  <div
    v-if="showPrompt"
    class="absolute left-0 top-0 h-screen w-screen flex flex-col justify-center items-center"
    style="width:100dvw;height:100dvh;background-size: cover;"
    :style="{
      'background-image': `url('/kaleidoscope-${Math.ceil(Math.random() * 9)}.jpg')`
    }"
  >
    <Prompt
      @click:enter="dismissPrompt"
    />
  </div>
</template>

<style scoped>
</style>
