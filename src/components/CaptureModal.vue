<script setup lang="ts">

defineProps<{
  url: string
}>();

const emit = defineEmits(['done']);

</script>

<template>
  <div
    class="
      m-auto
      flex
      flex-col
      items-center
      gap-4
      bg-white dark:bg-neutral-800
      bg-opacity-100 dark:bg-opacity-70
      backdrop-blur-xl
      rounded-3xl
      p-4
      shadow-xl
      w-7/8
      max-w-96
      pointer-events-auto
      ring-1
      ring-neutral-800/10
      select-text
    "
    @click.stop
  >
    <img
      class="w-full aspect-square rounded-lg pointer-events-auto touch-auto dark:shadow-[inset_0_0_1px_1px_rgba(255,255,255,0.9)]"
      style="-webkit-touch-callout: default;"
      alt="Captured kaleidoscope image."
      :src="url"
    >
    <p
      class="
        max-w-56 sm:max-w-full
        text-xs text-center
        text-black/60 dark:text-white/60
        select-text
      "
    >
      Right-click (or long-press) to download and share this image.<br>
      Otherwise, <a
        :href="url"
        target="_blank"
        class="
          text-blue-600 dark:text-blue-300
          select-text
        "
        download="kaleidoscope.jpg"
      >click here to download</a>.
    </p>
    <button
      class="bg-neutral-200 active:bg-neutral-300 dark:bg-neutral-100/20 dark:active:bg-neutral-200/20 font-bold text-black dark:text-white rounded-lg px-4 py-2 cursor-pointer w-full flex flex-row items-center gap-2 justify-center basis-1/4"
      @click="emit('done')"
    >
      Done
    </button>
  </div>
</template>

<style scoped>

</style>
