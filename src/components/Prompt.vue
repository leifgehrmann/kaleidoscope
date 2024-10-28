<script setup lang="ts">

const emit = defineEmits(['click:enter']);

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
    "
  >
    <h1 class="text-2xl font-bold">
      Kaleidoscope
    </h1>
    <p>Use your camera to create art from your environment.</p>
    <div class="flex justify-end">
      <button
        class="bg-white active:bg-neutral-100 font-bold text-blue-500 rounded-lg px-4 py-1 w-fit cursor-pointer"
        @click="emit('click:enter')"
      >
        Play
      </button>
    </div>
  </div>
</template>

<style scoped>

</style>
