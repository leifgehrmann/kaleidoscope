<script setup lang="ts">

import {ref} from 'vue';

const emit = defineEmits(['click:enter']);
const showInformation = ref(false);

</script>

<template>
  <div
    class="
      flex
      flex-col
      gap-2
      bg-white
      dark:bg-neutral-800
      bg-opacity-80
      dark:bg-opacity-70
      backdrop-blur-xl
      rounded-3xl
      p-4
      shadow-xl
      max-w-72
      pointer-events-auto
      ring-1
      ring-neutral-800/10
      transition-all
      select-text
    "
  >
    <h1 class="text-2xl font-bold select-text">
      Kaleidoscope
    </h1>
    <p class="select-text">
      Use your camera to create art from your environment.
    </p>
    <div
      class="flex gap-4"
      :class="{
        'justify-between': !showInformation,
        'justify-end': showInformation,
      }"
    >
      <button
        v-if="!showInformation"
        class="
          bg-white/0 text-black/70 active:text-black/60 dark:text-white/70 dark:active:text-white/60
          rounded-lg px-0 py-1 w-fit
        "
        @click="showInformation = !showInformation"
      >
        More information…
      </button>
      <button
        class="bg-blue-500 active:bg-blue-400 font-bold text-white rounded-lg px-4 py-1 w-fit cursor-pointer"
        @click="emit('click:enter')"
      >
        Play
      </button>
    </div>
    <Transition
      enter-active-class="transition-[opacity_max-height] duration-500"
      enter-from-class="opacity-0 max-h-0"
      enter-to-class="opacity-100 max-h-40"
    >
      <p
        v-if="showInformation"
        class="pt-2 text-xs overflow-hidden select-text"
      >
        This website does not collect any data. Additional information for this website, including its source code, can be found on
        <a
          href="https://github.com/leifgehrmann/kaleidoscope"
          class="text-blue-600 dark:text-blue-300 select-text"
        >GitHub</a>.
      </p>
    </Transition>
  </div>
</template>

<style scoped>

</style>
