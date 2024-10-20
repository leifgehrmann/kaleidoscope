<script setup lang="ts">
import {ref, onMounted, useTemplateRef} from 'vue';

const props = defineProps<{
  scopeShape: number // Todo replace with enum
}>();

const facingMode = ref('unknown');
const scopeRotation = ref(0.0);
const scopeSize = ref(0.5);
const scopeOffset = ref([0.0, 0.0]);
const canvas = useTemplateRef('canvas');

async function main() {
  // Capture webcam input using invisible `video` element
  // Adapted from p5js.org/examples/3d-shader-using-webcam.html
  const camera = document.getElementById('camera') as HTMLVideoElement;

  // Ask user permission to record their camera
  let stream: MediaStream | null = null;
  try {
    stream = await navigator.mediaDevices.getUserMedia({video: { facingMode: { exact: 'environment'} }, audio: false});
    facingMode.value = stream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
  } catch (e) {
    console.log('Failed to get environment camera. Trying any camera, under the assumption it is a user-facing camera', e);
    try {
      stream = await navigator.mediaDevices.getUserMedia({video: true, audio: false});
      facingMode.value = stream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
    } catch (e2) {
      if ((e2 as Error).name === 'ConstraintNotSatisfiedError') {
        console.log('Device has no camera', e2);
      } else if ((e2 as Error).name === 'PermissionDeniedError') {
        console.log('Permissions not accepted', e2);
      } else {
        console.log('Other error', e2);
      }
    }
  }

  if (stream !== null) {
    camera.srcObject = stream;
    camera.play();
    console.log('facingMode:', facingMode);
  }

  // Canvas with WebGL context
  const canvas = document.getElementById('maincanvas') as HTMLCanvasElement;
  const canvasSize = 1024;
  const gl = canvas.getContext('webgl')!;
  canvas.width = canvas.height = canvasSize;
  gl.viewport(0, 0, canvas.width, canvas.height);

  // Vertex shader: Identity map
  const vshader = gl.createShader(gl.VERTEX_SHADER)!;
  gl.shaderSource(vshader,
      'attribute vec2 p;'+
      'void main(){'+
      '    gl_Position = vec4(p,0,1);'+
      '}');
  gl.compileShader(vshader);

  // Fragment shader: sample video texture, change colors
  const fshader = gl.createShader(gl.FRAGMENT_SHADER)!;
  gl.shaderSource(fshader,`
      precision mediump float;

      uniform sampler2D data;
      uniform vec2 dataDimensions;
      uniform int dataIsFacingUser;
      uniform vec2 canvasDimensions;
      uniform int scopeShape;
      uniform float scopeRotation;
      uniform float scopeSize;
      uniform vec2 scopeOffset;

      float round(float v) {
          return floor(v + 0.5);
      }

      vec2 rotate2d(vec2 u, float theta) {
        return vec2(
          cos(theta) * u.x - sin(theta) * u.y,
          sin(theta) * u.x + cos(theta) * u.y
        );
      }

      // https://observablehq.com/@jrus/hexround
      vec2 axial_round(vec2 pos) {
        float xGrid = round(pos.x);
        float yGrid = round(pos.y);
        pos.x -= xGrid; // remainder
        pos.y -= yGrid; // remainder
        float dx = round(pos.x + 0.5*pos.y) * (pos.x*pos.x >= pos.y*pos.y ? 1.0 : 0.0);
        float dy = round(pos.y + 0.5*pos.x) * (pos.x*pos.x < pos.y*pos.y ? 1.0 : 0.0);
        return vec2(xGrid + dx, yGrid + dy);
      }

      vec2 square_float_to_axial_hex_grid(vec2 pos, bool pointyTop) {
        float SQRT3 = sqrt(3.0);
        float ratio = 2.0 / 3.0;
        if (pointyTop) {
          pos.x = ratio * 0.5 * (SQRT3*pos.x - pos.y);
          pos.y *= ratio;
        } else {
          pos.y = ratio * 0.5 * (SQRT3*pos.y - pos.x);
          pos.x *= ratio;
        }
        return axial_round(pos);
      }

      vec2 hexToCentroid(vec2 hex, bool pointyTop) {
        vec2 pos = vec2(0.0,0.0);
        float size = 2.0 / 3.0;
        if (pointyTop) {
          pos.y = hex.y / size;
          pos.x = hex.x * 2.0 / sqrt(3.0) / size + hex.y / sqrt(3.0) / size;
        } else {
          pos.x = hex.x / size;
          pos.y = hex.y * 2.0 / sqrt(3.0) / size + hex.x / sqrt(3.0) / size;
        }
        return pos;
      }

      vec2 square(vec2 u, float kLength, float kRot, vec2 offset) {

        u -= vec2(0.5, 0.5);
        u = rotate2d(u,kRot);
        u /= kLength;
        u += vec2(0.5, 0.5);

        // Center the square in a circle
        u.x *= sqrt(2.0);
        u.y *= sqrt(2.0);
        u += vec2((1.0-sqrt(2.0))/2.0, (1.0-sqrt(2.0))/2.0);

        u += offset;

        float kOffset = (1.0/1.0 - 1.0) / (1.0/1.0 * 2.0);
        vec2 k = vec2(0.0, 0.0);
        if (mod(-kOffset + u.x, 1.0 * 2.0) > 1.0) {
          k.x = 1.0 - mod(-kOffset + u.x, 1.0) / 1.0;
        } else {
          k.x = mod(-kOffset + u.x, 1.0) / 1.0;
        }
        if (mod(-kOffset + u.y, 1.0 * 2.0) > 1.0) {
          k.y = 1.0 - mod(-kOffset + u.y, 1.0) / 1.0;
        } else {
          k.y = mod(-kOffset + u.y, 1.0) / 1.0;
        }
        return k;
      }

      vec2 equilateral(vec2 u, float kLength, float kRot, vec2 offset) {
        // Center the triangle in a circle

        u -= vec2(0.5, 0.5);

        u = rotate2d(u,kRot);
        u /= kLength;
        u /= sqrt(3.0) / 2.0;

        u += vec2(0.5, 0.5);
        u.y -= (0.25) * sqrt(3.0) / 2.0;

        u += offset;

        vec2 k = vec2(0.0, 0.0);

        vec2 hexIndex = square_float_to_axial_hex_grid(u, false);
        vec2 hexCentroid = hexToCentroid(hexIndex, false);
        // For debugging
        // k = vec2(distance(hexToCentroid(hexIndex, false), u));
        // k = hexIndex / 5.0;

        float distance = distance(hexCentroid, u);
        float deg180 = 3.1415926536;
        float deg60 = deg180 / 3.0;
        float theta = atan(u.x- hexCentroid.x, u.y- hexCentroid.y) + deg180 + deg60 / 2.0;
        if (mod(theta, deg60 * 2.0) > deg60) {
          theta = mod(theta, deg60);
        } else {
          theta = deg60 - mod(theta, deg60);
        }

        k.x = cos(theta) * distance;
        k.y = sin(theta) * distance;

        // We want the center of the equilateral triangle to be the center of the image.
        k *= cos(radians(30.0));
        k.x += (((1.0 - sqrt(3.0) / 2.0)) / 2.00);
        k.y += (0.254); // I don't know what number this actually is meant to be... But it works.

        return k;
      }

      vec2 isosceles(vec2 u, float kLength, float kRot, vec2 offset) {
        // Center the triangle in a circle
        u -= vec2(0.5, 0.5);

        u = rotate2d(u,kRot);
        u /= kLength;
        u *= sqrt(2.0) / 2.0;

        u += vec2(0.5, 0.5);
        u -= 0.25;

        u += offset;

        vec2 k = vec2(0.0, 0.0);

        vec2 squareCentroid = vec2(round(u.x), round(u.y));
        // For debugging
        // k = vec2(distance(squareCentroid, u));

        float distance = distance(squareCentroid, u) * 2.0;
        float deg180 = 3.1415926536;
        float deg45 = deg180 / 4.0;
        // Multiply by 0.99999, because atan2 fails on some corner cases :(
        float theta = atan(u.x- squareCentroid.x, u.y- squareCentroid.y) * 0.999999;
        if (mod(theta, deg45 * 2.0) > deg45) {
          theta = deg45 - mod(theta, deg45);
        } else {
          theta = mod(theta, deg45);
        }

        k.x = cos(theta) * distance;
        k.y = sin(theta) * distance;

        return k;
      }

      vec2 scalene(vec2 u, float kLength, float kRot, vec2 offset) {
        u -= 0.5;

        u = rotate2d(u,kRot + radians(60.0));
        u /= kLength;

        u += 0.5;

        u.x -= sin(radians(90.0)) * 0.50;
        u.y -= cos(radians(90.0)) * 0.50;

        u += offset;

        vec2 k = vec2(0.0, 0.0);

        vec2 hexIndex = square_float_to_axial_hex_grid(u, true);
        vec2 hexCentroid = hexToCentroid(hexIndex, true);
        // For debugging
        // k = vec2(distance(hexToCentroid(hexIndex, true), u));
        // k = hexIndex / 5.0;

        float distance = distance(hexCentroid, u) * 2.0 / sqrt(3.0);
        float deg180 = 3.1415926536;
        float deg30 = deg180 / 6.0;
        float theta = atan(u.x- hexCentroid.x, u.y- hexCentroid.y) + deg180;
        if (mod(theta, deg30 * 2.0) > deg30) {
          theta = mod(theta, deg30);
        } else {
          theta = deg30 - mod(theta, deg30);
        }

        k.x = cos(theta) * distance;
        k.y = sin(theta) * distance;

        // We want the center of the triangle to be the center of the image.
        k *= sqrt(3.0) / 2.0;
        k.x += (((1.0 - sqrt(3.0) / 2.0)) / 2.0);
        k.y += (0.25);

        return k;
      }

      void main() {
          // The interesting things to change!
          float kLength = scopeSize; // Effectively, how often should the image reflect 1 is one reflection (when viewed in square mode)
          float dataScopePercentage = 1.0; // Effectively how narrow the kaleidoscope should be, as a percentage of the camera size

          vec2 centerOffset = vec2(0.0);

          vec2 fragCoord = vec2(
            gl_FragCoord.x / canvasDimensions.x,
            gl_FragCoord.y / canvasDimensions.y
          );

          // Position in kaleidoscope space
          // In opengl, y-coordinate is flipped
          vec2 u = vec2(fragCoord.x,1.0-fragCoord.y);
          vec2 k = u;

          vec2 scopeOrigin = vec2(0.0, 0.0);
          float scopeDiameterRatio = sqrt(2.0);
          vec2 d = rotate2d(scopeOffset, 0.0);
          if (scopeShape == 0) {
            k = equilateral(k, scopeSize, scopeRotation, d);
            scopeDiameterRatio = 1.0;
          } else if (scopeShape == 1) {
            k = square(k, scopeSize, scopeRotation, d);
            scopeDiameterRatio = sqrt(2.0);
          } else if (scopeShape == 2) {
            k = isosceles(k, scopeSize, scopeRotation, d);
          } else {
            scopeDiameterRatio = 1.0;
            k = scalene(k, scopeSize, scopeRotation, d);
          }

          // Now map the k value to coordinates on the image
          // 0,0 will be the centre of the image
          // 1,1 will be the top right of the image (not the bottom left– It's easier to orientate if things are up-right)
          float dataMinDimension = min(dataDimensions.x, dataDimensions.y) / scopeDiameterRatio;
          float dataWindowSize = dataMinDimension * dataScopePercentage;
          vec2 i = vec2(0.0,0.0);
          // x-axis is flipped only when the camera is pointed to the user
          i.x = (-dataWindowSize / 2.0 + k.x * dataWindowSize) * (dataIsFacingUser == 1 ? -1.0 : 1.0);
          // y-axis is flipped because of openGL coordinate space
          i.y = - (-dataWindowSize / 2.0 + k.y * dataWindowSize);
          i = rotate2d(i, scopeRotation * (dataIsFacingUser == 1 ? -1.0 : 1.0));
          i.x += dataDimensions.x / 2.0;
          i.y += dataDimensions.y / 2.0;

          gl_FragColor=texture2D(data,vec2(i.x, i.y)/vec2(dataDimensions.x,dataDimensions.y)).xyzw;

          // For debugging the kaleidoscope value
          // gl_FragColor=vec4(k.x, k.y, 0.0, 1.0);
      }`);
  gl.compileShader(fshader);

  // Create and link program
  const program  = gl.createProgram()!;
  gl.attachShader(program,vshader);
  gl.attachShader(program,fshader);
  gl.linkProgram(program);
  gl.useProgram(program);

  // Vertices: A screen-filling quad made from two triangles
  gl.bindBuffer(gl.ARRAY_BUFFER, gl.createBuffer());
  gl.bufferData(gl.ARRAY_BUFFER,new Float32Array([-1,-1,1,-1,-1,1,-1,1,1,-1,1,1]),gl.STATIC_DRAW);
  gl.enableVertexAttribArray(0);
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0);

  // Texture to contain the video data
  const texture = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, texture);
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);

  // Bind texture to the "data" argument to the fragment shader
  gl.uniform1i(gl.getUniformLocation(program,'data'),0);
  gl.activeTexture(gl.TEXTURE0);
  gl.bindTexture(gl.TEXTURE_2D,texture);

  // Bind camera dimensions to the fragment shader
  const dataDimensionsBind = gl.getUniformLocation(program, 'dataDimensions');
  const dataIsFacingUserBind = gl.getUniformLocation(program, 'dataIsFacingUser');
  const canvasDimensionsBind = gl.getUniformLocation(program, 'canvasDimensions');
  const scopeShapeBind = gl.getUniformLocation(program, 'scopeShape');
  const scopeRotationBind = gl.getUniformLocation(program, 'scopeRotation');
  const scopeSizeBind = gl.getUniformLocation(program, 'scopeSize');
  const scopeOffsetBind = gl.getUniformLocation(program, 'scopeOffset');

  // Repeatedly pull camera data and render
  function animate(){
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, camera);
    gl.uniform2f(dataDimensionsBind, camera.videoWidth, camera.videoHeight);
    gl.uniform1i(dataIsFacingUserBind, facingMode.value === 'user' ? 1 : 0);
    gl.uniform1i(scopeShapeBind, props.scopeShape);
    let scopeRotationOffset = 0;
    if (props.scopeShape === 0) {
      scopeRotationOffset = Math.PI / 3;
    } else if (props.scopeShape === 2) {
      scopeRotationOffset = -Math.PI / 2;
    } else if (props.scopeShape === 3) {
      scopeRotationOffset = -Math.PI / 2;
    }
    gl.uniform1f(scopeRotationBind, scopeRotation.value + scopeRotationOffset);
    gl.uniform1f(scopeSizeBind, scopeSize.value);
    gl.uniform2f(scopeOffsetBind, scopeOffset.value[0], scopeOffset.value[1]);
    gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);
    gl.drawArrays(gl.TRIANGLES, 0, 6);
    requestAnimationFrame(animate);
  }
  animate();
}

interface Point {
  clientX: number;
  clientY: number;
}
let touchId1: null|number = null;
let touchPrev1: null|Point = null;

function getTouchById(touches: TouchList, id: number): Touch | null {
  for (let i = 0; i < touches.length; i += 1) {
    if (touches[i].identifier === id) {
      return touches[i];
    }
  }
  return null;
}

function touchStartCallback(event: TouchEvent) {
  event.preventDefault();
  if (touchId1 !== null) {
    return;
  }
  const touch = event.changedTouches[0];
  touchId1 = touch.identifier;
  touchPrev1 = touch;
}

function touchMoveCallback(event: TouchEvent) {
  if (touchId1 === null || touchPrev1 === null) {
    return;
  }
  const touch = getTouchById(event.changedTouches, touchId1);
  if (touch === null) {
    return;
  }
  const deltaX = touch.clientX - touchPrev1.clientX;
  const deltaY = touch.clientY - touchPrev1.clientY;
  scopeRotation.value += deltaX / 100;
  scopeSize.value *= 1.0 + deltaY / 100;
  touchPrev1 = touch;
}

function touchEndCallback(event: TouchEvent) {
  if (touchId1 === null || touchPrev1 === null) {
    return;
  }
  const touch = getTouchById(event.changedTouches, touchId1);
  if (touch === null) {
    return;
  }
  touchId1 = null;
  touchPrev1 = null;
}

onMounted(() => {
  main();

  const canvasElement = canvas.value as HTMLCanvasElement;

  let mousePrevPosition = null as null|{x: number, y: number};
  canvasElement.addEventListener('mousedown', (mouseEvent) => {
    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
  });
  document.addEventListener('mousemove', (mouseEvent) => {
    if (mousePrevPosition === null) {
      return;
    }
    const deltaX = mouseEvent.clientX - mousePrevPosition.x;
    const deltaY = mouseEvent.clientY - mousePrevPosition.y;
    scopeRotation.value += deltaX / 100;
    scopeSize.value *= 1.0 + deltaY / 100;
    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
  });
  document.addEventListener('mouseup', (mouseEvent) => {
    if (mousePrevPosition === null) {
      return;
    }
    // const delta = {
    //   dx: mouseEvent.clientX - mousePrevPosition.x,
    //   dy: mouseEvent.clientY - mousePrevPosition.y,
    // };
    // scopeRotation.value += delta.dx / 100;
    // scopeSize.value *= 1.0 - delta.dy / 100;
    mousePrevPosition = null;
  });

  canvasElement.addEventListener('touchstart', touchStartCallback);
  canvasElement.addEventListener('touchmove', touchMoveCallback);
  canvasElement.addEventListener('touchend', touchEndCallback);
});

document.addEventListener('keydown', (keyEvent) => {
  if (keyEvent.key == 'a') {
    scopeRotation.value -= 0.1;
  } else if (keyEvent.key == 'd') {
    scopeRotation.value += 0.1;
  }

  if (keyEvent.key == 'w') {
    scopeSize.value *= 0.9;
  } else if (keyEvent.key == 's') {
    scopeSize.value *= 1.1;
  }

  if (keyEvent.code == 'ArrowDown') {
    scopeOffset.value[0] += Math.sin(scopeRotation.value) * -0.1;
    scopeOffset.value[1] += Math.cos(scopeRotation.value) * -0.1;
  } else if (keyEvent.key == 'ArrowUp') {
    scopeOffset.value[0] += Math.sin(scopeRotation.value) * 0.1;
    scopeOffset.value[1] += Math.cos(scopeRotation.value) * 0.1;
  } else if (keyEvent.key == 'ArrowLeft') {
    scopeOffset.value[0] += Math.cos(scopeRotation.value) * -0.1;
    scopeOffset.value[1] += Math.sin(scopeRotation.value) * -0.1;
  } else if (keyEvent.key == 'ArrowRight') {
    scopeOffset.value[0] += Math.cos(scopeRotation.value) * 0.1;
    scopeOffset.value[1] += Math.sin(scopeRotation.value) * 0.1;
  }
  const scopeOffsetDistance = Math.sqrt(scopeOffset.value[0] * scopeOffset.value[0] + scopeOffset.value[1] * scopeOffset.value[1])
  if (scopeOffsetDistance > 1) {
    scopeOffset.value[0] /= scopeOffsetDistance;
    scopeOffset.value[1] /= scopeOffsetDistance;
  }
});

</script>

<template>
  <canvas
    id="maincanvas"
    ref="canvas"
    style="width:100dvw;height:100dvh;object-fit:cover"
  />
  <video
    id="camera"
    visible="False"
    style="width: 512px; height: 512px; display:none;"
    controls="true"
    playsinline
    crossorigin="anonymous"
  />
</template>

<style scoped>
</style>
