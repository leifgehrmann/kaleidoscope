<script setup lang="ts">

import {ref} from 'vue';

const emit = defineEmits(['click:enter']);
const showInformation = ref(false);

</script>

<template>
  <div
    class="
      m-1
      flex
      flex-col
      items-center
      gap-6
      bg-white
      dark:bg-neutral-800
      bg-opacity-100
      dark:bg-opacity-90
      backdrop-blur-xl
      rounded-3xl
      p-4
      py-8
      shadow-xl
      sm:w-[24rem]
      sm:min-w-72
      pointer-events-auto
      ring-1
      ring-neutral-800/10
      select-text
    "
  >
    <h1 class="text-3xl font-bold select-text text-center">
      Kaleidoscope
    </h1>
    <ul class="max-w-64 flex flex-col gap-6">
      <li class="list-item">
        <img src="/public/equilateral.svg">
        <p>Use your camera to create art from your environment.</p>
      </li>
      <li class="list-item">
        <img src="/public/equilateral.svg">
        <p>
          Drag the screen to zoom and rotate the kaleido&shy;scope.
        </p>
      </li>
      <li class="list-item">
        <img src="/public/equilateral.svg">
        <p>
          Switch mirror arrangements to create different patterns.<br>
          <a
            href="https://github.com/leifgehrmann/kaleidoscope"
            target="_blank"
            class="text-blue-600 dark:text-blue-300 select-text"
          >Read more...</a>
        </p>
      </li>
    </ul>
    <p class="max-w-64 select-text text-center dark:text-white/60 text-xs">
      This website does not collect any data.<br>Additional information can be found on
      <a
          href="https://github.com/leifgehrmann/kaleidoscope"
          class="text-blue-600/80 dark:text-blue-300/80 select-text"
      >GitHub</a>.
    </p>
    <button
      class="bg-blue-500 active:bg-blue-400 font-bold text-white rounded-lg px-4 py-2 cursor-pointer w-full max-w-52"
      @click="emit('click:enter')"
    >
      Play
    </button>
  </div>
</template>

<style scoped>
.list-item {
  @apply flex gap-6;
}

.list-item p {
  @apply max-w-44 sm:max-w-52 select-text text-sm;
}
</style>
