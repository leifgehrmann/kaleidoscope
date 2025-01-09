<script setup lang="ts">

import {ref} from 'vue';

const { isResumePrompt = false } = defineProps<{
  isResumePrompt?: boolean
}>();

const emit = defineEmits(['click:enter']);
const showInformation = ref(false);

</script>

<template>
  <div
    class="
      m-auto
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
        <picture>
          <source
            srcset="/prompt-camera-dark.svg"
            media="(prefers-color-scheme: dark)"
          >
          <img
            src="/prompt-camera-light.svg"
            alt="A camera icon."
          >
        </picture>
        <p>Use your camera to create art from your environment.</p>
      </li>
      <li class="list-item">
        <picture>
          <source
            srcset="/prompt-drag-dark.svg"
            media="(prefers-color-scheme: dark)"
          >
          <img
            src="/prompt-drag-light.svg"
            alt="A finger dragging from right to left."
          >
        </picture>
        <p>
          Drag the screen to rotate and zoom the kaleido&shy;scope.
        </p>
      </li>
      <li class="list-item">
        <picture>
          <source
            srcset="/prompt-shapes-dark.svg"
            media="(prefers-color-scheme: dark)"
          >
          <img
            src="/prompt-shapes-light.svg"
            alt="4 geometric shapes."
          >
        </picture>
        <p>
          Switch mirror arrangements to create different patterns.<br>
          <button
            class="inline-block text-blue-600 dark:text-blue-300 select-text"
            @click="showInformation = !showInformation"
          >
            <span v-if="!showInformation">Learn more…</span>
            <span v-else>Show less…</span>
          </button>
        </p>
      </li>
    </ul>
    <Transition
      enter-active-class="transition-[opacity_max-height] duration-500 ease-out"
      enter-from-class="opacity-0 max-h-0"
      enter-to-class="opacity-100 max-h-72"
    >
      <div
        v-if="showInformation"
        class="
          max-w-64 md:max-w-72 p-4 rounded-xl flex flex-col gap-2
          bg-neutral-100
          dark:bg-neutral-900
          text-xs text-black/80 dark:text-white/90
          overflow-hidden
        "
      >
        <p>Kaleidoscopes are created by tilting two or more mirrors towards each other at an angle.</p>
        <p>This website features kaleidoscopes that use 3 or 4 mirrors, also known as &lsquo;Polycentral Kaleidoscopes&rsquo;, which create infinitely repeating patterns.</p>
        <p>
          Only 4 shapes can create a tessellating pattern. For why this is, see Wikipedia's article on <a
            href="https://en.wikipedia.org/wiki/Edge_tessellation"
            target="_blank"
          >&lsquo;Edge Tesselation&rsquo;</a>.
        </p>
        <p>
          For additional information, check out my post <a
            href="https://leifgehrmann.com/kaleidoscopes/"
            target="_blank"
          >&lsquo;Digital Kaleidoscopes&rsquo;</a> on my personal website.
        </p>
      </div>
    </Transition>
    <p class="max-w-64 select-text text-center text-black/60 dark:text-white/60 text-xs">
      This website does not collect any data.<br>Additional information can be found on
      <a
        href="https://github.com/leifgehrmann/kaleidoscope"
        class="text-blue-600/80 dark:text-blue-300/80 select-text"
      >GitHub</a>.
    </p>
    <button
      class="bg-blue-600 active:bg-blue-500 dark:bg-blue-500 dark:active:bg-blue-400 font-bold text-white rounded-lg px-4 py-2 cursor-pointer w-full max-w-52"
      @click="emit('click:enter')"
    >
      <span v-if="isResumePrompt">Resume</span>
      <span v-else>Play</span>
    </button>
  </div>
</template>

<style scoped>
p {
  @apply select-text;
}

p a {
  @apply text-blue-600 dark:text-blue-300 select-text;
  -webkit-touch-callout: default;
}

.list-item {
  @apply flex gap-4 items-center;
}

.list-item img {
  @apply w-8;
}

.list-item p {
  @apply max-w-44 sm:max-w-52 select-text text-sm;
}
</style>
