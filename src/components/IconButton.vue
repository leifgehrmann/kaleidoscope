<script setup lang="ts">
import {onMounted, ref, useTemplateRef} from 'vue';

defineProps<{
  label: string,
  imageUrl: string
}>();

const emit = defineEmits(['press']);

const pressActive = ref(false);
const buttonRef = useTemplateRef('button');

onMounted(() => {
  const button = buttonRef.value as HTMLButtonElement;
  let originatedFromPointerdown = false;
  button.addEventListener('pointerdown', () => {
    pressActive.value = true;
  });
  button.addEventListener('pointerenter', () => {
    if (originatedFromPointerdown) {
      pressActive.value = true;
    }
    originatedFromPointerdown = false;
  });
  button.addEventListener('pointerleave', () => {
    if (pressActive.value) {
      originatedFromPointerdown = true;
    }
    pressActive.value = false;
  });
  button.addEventListener('pointerout', () => {
    if (pressActive.value) {
      originatedFromPointerdown = true;
    }
    pressActive.value = false;
  });
  button.addEventListener('pointerup', () => {
    if (pressActive.value) {
      emit('press');
    }
  });
  button.addEventListener('click', () => {
    emit('press');
  });
  function reset() {
    pressActive.value = false;
    originatedFromPointerdown = false;
  }
  window.addEventListener('pointerup', reset);
  button.addEventListener('pointercancel', reset);
});
</script>

<template>
  <button
    ref="button"
    class="
      pointer-events-auto
      h-full rounded-lg
      flex justify-center items-center
      cursor-pointer
    "
    :aria-label="label"
    :title="label"
  >
    <img
      class="
        h-full
        pointer-events-none
        transition-opacity ease-out duration-300
      "
      :class="{
        'opacity-100 dark:opacity-70': !pressActive,
        'opacity-80 dark:opacity-60': pressActive
      }"
      aria-hidden="true"
      :src="imageUrl"
      alt=""
    >
  </button>
</template>

<style scoped>

</style>
