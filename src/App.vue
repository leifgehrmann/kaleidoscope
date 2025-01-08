<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import SegmentedControl, { Option } from './components/SegmentedControl.vue';
import {onUpdated, ref, useTemplateRef} from 'vue';
import SteeringControl from './components/SteeringControl.vue';
import { ScopeShape } from './scopeShape.ts';
import Prompt from './components/Prompt.vue';
import IconButton from './components/IconButton.vue';

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
const showInfo = ref(false);
const showControls = ref(true);
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
  showInfo.value = false;
}

window.addEventListener('keypress', (keyEvent) => {
  if (keyEvent.code == 'Digit1') {
    selectedIndex.value = ScopeShape.Equilateral;
  } else if (keyEvent.code == 'Digit2') {
    selectedIndex.value = ScopeShape.Square;
  } else if (keyEvent.code == 'Digit3') {
    selectedIndex.value = ScopeShape.Isosceles;
  } else if (keyEvent.code == 'Digit4') {
    selectedIndex.value = ScopeShape.Scalene;
  }
  if (keyEvent.code != 'KeyH') {
    return;
  }
  if (showPrompt.value) {
    return;
  }
  showControls.value = !showControls.value;
});

const controlsBarRef = useTemplateRef('controls-bar');

onUpdated(() => {
  const controlsBar = controlsBarRef.value as HTMLDivElement | null;
  if (controlsBar !== null) {
    controlsBar.addEventListener('touchstart', (e) => { e.preventDefault(); });
    controlsBar.addEventListener('touchmove', (e) => { e.preventDefault(); });
  }
});

</script>

<template>
  <Kaleidoscope
    v-if="!showPrompt"
    :scope-shape="selectedIndex"
    :scope-auto-rotation-velocity="scopeAutoRotationVelocity"
  />
  <div
    v-if="!showPrompt && !showInfo && showControls"
    class="absolute w-full flex flex-col justify-end gap-1 px-2 py-1 items-center pointer-events-none"
    style="bottom:calc(env(safe-area-inset-bottom))"
  >
    <SteeringControl
      class="w-full max-w-56 h-6"
      @update:velocity="updateScopeAutoRotationVelocity"
    />
    <div
      ref="controls-bar"
      class="
      pointer-events-none
      w-full max-w-64 flex flex-row gap-1
      items-center
      p-1
      relative
      bg-neutral-300
      bg-opacity-50
      dark:bg-neutral-800
      dark:bg-opacity-70
      shadow-lg
      dark:shadow-none
      backdrop-blur-xl
      rounded-xl
      text-sm
      overflow-hidden
      "
    >
      <SegmentedControl
        class="w-full pointer-events-auto"
        :options="options"
        :selected-index="selectedIndex"
        @update:selected-index="updateSelectedIndex"
      />
      <div
        class="bg-white/30 rounded-full w-0.5 h-6"
      ></div>
      <div
        class="flex pointer-events-auto"
      >
        <IconButton
          image-url="/capture.svg"
          label="Capture"
        />
        <IconButton
          image-url="/info.svg"
          label="Info"
          @press="showInfo = true"
        />
      </div>
    </div>
  </div>
  <div
    v-if="showInfo"
    class="absolute left-0 top-0 h-screen w-screen grid grid-cols-1 grid-rows-1 p-2 overflow-auto"
    style="width:100dvw;height:100dvh;background-size: cover;"
  >
    <Prompt
      @click:enter="dismissPrompt"
    />
  </div>
  <div
    v-if="showPrompt"
    class="absolute left-0 top-0 h-screen w-screen grid grid-cols-1 grid-rows-1 p-2 overflow-auto"
    style="width:100dvw;height:100dvh;background-size: cover;"
    :style="{
      'background-image': `url('/kaleidoscope-${Math.ceil(Math.random() * 10)}.jpg')`
    }"
  >
    <Prompt
      @click:enter="dismissPrompt"
    />
  </div>
</template>

<style scoped>
</style>
