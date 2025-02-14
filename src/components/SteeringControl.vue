<script setup lang="ts">

import {computed, onMounted, ref, useTemplateRef, watch} from 'vue';

interface Circle {
  x: number,
  y: number,
  r: number
}

const emit = defineEmits(['update:velocity']);

const resizeObserver = ref(null as null|ResizeObserver);
const circleLeft = ref({x: 0, y: 0, r: 1} as Circle);
const circleRight = ref({x: 0, y: 0, r: 1} as Circle);
const circleLeftTarget = ref({x: 0, y: 0, r: 1} as Circle);
const circleRightTarget = ref({x: 0, y: 0, r: 1} as Circle);
const isSteering = ref(false);
const hasChangedSteeringAfterFirstPress = ref(false);
const steerDirection = ref(0); // -1 | 0 | 1
const steerVelocity = ref(0);
const pointerPressed = ref(false);
const pointerOrigin = ref(null as null | Touch | MouseEvent);
const container = useTemplateRef('container');
const touchSurface = useTemplateRef('touchSurface');

const joiningPolygon = computed(() => {
  const gamma = Math.atan2(
    circleRight.value.y - circleLeft.value.y,
    circleRight.value.x - circleLeft.value.x,
  );
  const beta = Math.asin(
      (circleRight.value.r - circleLeft.value.r) /
      Math.sqrt((
          Math.pow(circleRight.value.x - circleLeft.value.x, 2) +
          Math.pow(circleRight.value.y - circleLeft.value.y, 2)
      ))
  );
  const alpha = gamma - beta;
  const tangentLeftA = {
    x: circleLeft.value.x + circleLeft.value.r * Math.sin(alpha),
    y: circleLeft.value.y + circleLeft.value.r * Math.cos(alpha),
  };
  const tangentRightA = {
    x: circleRight.value.x + circleRight.value.r * Math.sin(alpha),
    y: circleRight.value.y + circleRight.value.r * Math.cos(alpha),
  };
  const tangentLeftB = {
    x: circleLeft.value.x - circleLeft.value.r * Math.sin(-alpha),
    y: circleLeft.value.y - circleLeft.value.r * Math.cos(-alpha),
  };
  const tangentRightB = {
    x: circleRight.value.x - circleRight.value.r * Math.sin(-alpha),
    y: circleRight.value.y - circleRight.value.r * Math.cos(-alpha),
  };
  return `
    M ${circleLeft.value.x} ${circleLeft.value.y}
    L ${tangentLeftA.x} ${tangentLeftA.y}
    L ${tangentRightA.x} ${tangentRightA.y}
    L ${circleRight.value.x} ${circleRight.value.y}
    L ${tangentRightB.x} ${tangentRightB.y}
    L ${tangentLeftB.x} ${tangentLeftB.y}
    L ${circleLeft.value.x} ${circleLeft.value.y}
  `;
});

function getContainerElement(): null|HTMLDivElement {
  return container.value as null|HTMLDivElement;
}

function getTouchSurfaceElement(): null|HTMLDivElement {
  return touchSurface.value as null|HTMLDivElement;
}

function resizeCallback(entries: ResizeObserverEntry[]) {
  const containerElement = getContainerElement();
  entries.forEach((entry) => {
    if (containerElement === null) {
      return;
    }
    const center = entry.contentRect.width / 2;
    const radius = entry.contentRect.height / 2;
    const gap = radius * 0.25;
    circleLeft.value = {
      x: center - (radius + gap / 2),
      y: radius,
      r: radius
    };
    circleRight.value = {
      x: center + (radius + gap / 2),
      y: radius,
      r: radius
    };
  });
}

function calculatePivotRadius(value: number, maxRadius: number) {
  return maxRadius - 0.2 - Math.abs(asymptote(value, 1)) * maxRadius * 0.7;
}

function pointerStart(containerElement: HTMLDivElement, pointer: Touch | MouseEvent) {
  const rect = containerElement.getBoundingClientRect();
  const center = rect.width / 2;
  const radius = rect.height / 2;
  let mouseOffset = (pointer.clientX - (rect.x + center));
  mouseOffset += Math.sign(mouseOffset) * radius / 4;

  isSteering.value = true;
  hasChangedSteeringAfterFirstPress.value = true;
  steerDirection.value = Math.sign(mouseOffset);
  steerVelocity.value = mouseOffset / center;

  const valueLimit = center - radius * 1.75;
  const circleHandleTarget = mouseOffset > 0 ? circleRightTarget : circleLeftTarget;
  const circlePivotTarget = mouseOffset > 0 ? circleLeftTarget : circleRightTarget;

  circleHandleTarget.value.x = center + asymptote(mouseOffset, valueLimit);
  circleHandleTarget.value.y = radius;
  circleHandleTarget.value.r = radius;
  circlePivotTarget.value.x = center;
  circlePivotTarget.value.y = radius;
  circlePivotTarget.value.r = calculatePivotRadius(steerVelocity.value, radius);

  pointerPressed.value = true;
  pointerOrigin.value = pointer;

  animateCircles();
}

function pointerMove(containerElement: HTMLDivElement, pointer: Touch | MouseEvent) {
  if (!pointerPressed.value || pointerOrigin.value === null) {
    return;
  }
  const rect = containerElement.getBoundingClientRect();
  const center = rect.width / 2;
  const radius = rect.height / 2;
  let mouseOffset = (pointerOrigin.value.clientX - (rect.x + center)) + (pointer.clientX - pointerOrigin.value.clientX);
  mouseOffset += Math.sign(mouseOffset) * radius / 4;

  hasChangedSteeringAfterFirstPress.value = false;
  steerDirection.value = Math.sign(mouseOffset);
  steerVelocity.value = mouseOffset / center;

  const valueLimit = center - radius * 1.75;
  const circleHandle = mouseOffset > 0 ? circleRight : circleLeft;
  const circlePivot = mouseOffset > 0 ? circleLeft : circleRight;

  circleHandle.value.x = center + asymptote(mouseOffset, valueLimit);
  circleHandle.value.y = radius;
  circleHandle.value.r = radius;
  circlePivot.value.x = center;
  circlePivot.value.y = radius;
  circlePivot.value.r = calculatePivotRadius(steerVelocity.value, radius);
}

function pointerEnd(containerElement: HTMLDivElement) {
  pointerPressed.value = false;
  const rect = containerElement.getBoundingClientRect();
  const center = rect.width / 2;
  const radius = rect.height / 2;
  const gap = radius * 0.25;
  circleLeftTarget.value = {
    x: center - (radius + gap / 2),
    y: radius,
    r: radius
  };
  circleRightTarget.value = {
    x: center + (radius + gap / 2),
    y: radius,
    r: radius
  };
  isSteering.value = false;
  steerDirection.value = 0;
  steerVelocity.value = 0;
  animateCircles();
}

function animateCircles() {
  if (isSteering.value && !hasChangedSteeringAfterFirstPress.value) {
    return;
  }
  const circleLeftDeltaX = (circleLeftTarget.value.x - circleLeft.value.x) / 10;
  const circleLeftDeltaY = (circleLeftTarget.value.y - circleLeft.value.y) / 10;
  const circleLeftDeltaR = (circleLeftTarget.value.r - circleLeft.value.r) / 10;
  circleLeft.value.x += circleLeftDeltaX;
  circleLeft.value.y += circleLeftDeltaY;
  circleLeft.value.r += circleLeftDeltaR;

  const circleRightDeltaX = (circleRightTarget.value.x - circleRight.value.x) / 10;
  const circleRightDeltaY = (circleRightTarget.value.y - circleRight.value.y) / 10;
  const circleRightDeltaR = (circleRightTarget.value.r - circleRight.value.r) / 10;
  circleRight.value.x += circleRightDeltaX;
  circleRight.value.y += circleRightDeltaY;
  circleRight.value.r += circleRightDeltaR;

  const distanceSum = Math.abs(circleLeftDeltaX) +
      Math.abs(circleLeftDeltaY) +
      Math.abs(circleLeftDeltaR) +
      Math.abs(circleRightDeltaX) +
      Math.abs(circleRightDeltaY) +
      Math.abs(circleRightDeltaR)
  ;
  if (distanceSum > 0.01) {
    requestAnimationFrame(animateCircles);
  }
}

function getTouchById(touches: TouchList, id: number): Touch | null {
  for (let i = 0; i < touches.length; i += 1) {
    if (touches[i].identifier === id) {
      return touches[i];
    }
  }
  return null;
}

function asymptote(value: number, limit: number, sharpness = 5) {
  return value * limit / (Math.pow(Math.pow(limit, sharpness) + Math.pow(Math.abs(value), sharpness), 1 / sharpness));
}

watch(steerVelocity, () => {
  emit('update:velocity', steerVelocity.value);
});

onMounted(() => {
  const containerElement = getContainerElement();
  const touchSurfaceElement = getTouchSurfaceElement();
  if (containerElement === null || touchSurfaceElement === null) {
    return;
  }
  resizeObserver.value = new ResizeObserver(resizeCallback);
  resizeObserver.value.observe(containerElement);

  let touchId: null | number = null;

  touchSurfaceElement.addEventListener('touchstart', (touchEvent) => {
    touchEvent.preventDefault();
    if (touchId !== null) {
      return;
    }
    const touch = touchEvent.changedTouches[0];
    touchId = touch.identifier;
    pointerStart(containerElement, touch);
  });
  window.addEventListener('touchmove', (touchEvent) => {
    if (touchId === null) {
      return;
    }
    const touch = getTouchById(touchEvent.changedTouches, touchId);
    if (touch === null) {
      return;
    }
    pointerMove(containerElement, touch);
  });
  window.addEventListener('touchend', (touchEvent) => {
    if (touchId === null) {
      return;
    }
    const touch = getTouchById(touchEvent.changedTouches, touchId);
    if (touch === null) {
      return;
    }
    touchId = null;
    pointerEnd(containerElement);
  });

  touchSurfaceElement.addEventListener('mousedown', (mouseEvent) => {
    // Left mouse button only
    if (mouseEvent.button !== 0) {
      return;
    }
    pointerStart(containerElement, mouseEvent);
  });
  window.addEventListener('mousemove', (mouseEvent) => {
    pointerMove(containerElement, mouseEvent);
  });
  window.addEventListener('mouseup', () => {
    pointerEnd(containerElement);
  });
});

</script>

<template>
  <div ref="container">
    <div
      ref="touchSurface"
      class="
        h-full
        w-full
        bg-neutral-300/50
        dark:bg-neutral-800/70
        backdrop-blur-xl
        overflow-hidden
        shadow-md
        dark:shadow-none
        pointer-events-auto
        text-center
        cursor-pointer
      "
      :style="{
        'clip-path': 'url(#myClip)'
      }"
    >
      <svg
        class="transition-opacity w-full h-full"
        :class="{
          'opacity-0': isSteering && Math.abs(steerVelocity) < 0.05,
          'opacity-100': !isSteering || (steerDirection !== 0 && Math.abs(steerVelocity) >= 0.05)
        }"
      >
        <g
          v-if="!isSteering || steerDirection === -1"
          :transform="`translate(${circleLeft.x},${circleLeft.y}) scale(${1/10 * circleLeft.r * 0.6}) translate(-5,-5)`"
        >
          <path
            d="M 10,0 L 0,5 L 10,10"
            fill="white"
          />
        </g>
        <g
          v-if="!isSteering || steerDirection === 1"
          :transform="`translate(${circleRight.x},${circleLeft.y}) scale(${1/10 * circleRight.r * 0.6}) translate(-5,-5)`"
        >
          <path
            d="M 0,0 L 10,5 L 0,10"
            fill="white"
          />
        </g>
      </svg>
      <svg
        class="transition-opacity w-full h-full absolute top-0 left-0"
        :class="{
          'opacity-0': !isSteering || Math.abs(steerVelocity) > 0.05,
          'opacity-100': isSteering && Math.abs(steerVelocity) < 0.05,
        }"
      >
        <g
          :transform="`translate(${(circleLeft.x + circleRight.x)/2},${(circleLeft.y + circleRight.y)/2}) scale(${1/10 * circleLeft.r * 0.6})`"
        >
          <circle
            r="5"
            cx="0"
            cy="0"
            fill="white"
          />
        </g>
      </svg>
    </div>
    <svg
      width="0"
      height="0"
    >
      <defs>
        <clipPath id="myClip">
          <circle
            :cx="circleLeft.x"
            :cy="circleLeft.y"
            :r="circleLeft.r"
          />
          <circle
            :cx="circleRight.x"
            :cy="circleRight.y"
            :r="circleRight.r"
          />
          <path :d="joiningPolygon" />
        </clipPath>
      </defs>
    </svg>
  </div>
</template>

<style scoped>

</style>
