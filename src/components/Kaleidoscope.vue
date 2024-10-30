<script setup lang="ts">
import {ref, onMounted, useTemplateRef} from 'vue';
import { ScopeShape } from '../scopeShape.ts';

const props = defineProps<{
  scopeShape: ScopeShape,
  scopeAutoRotationVelocity: number
}>();

const facingMode = ref('unknown');
const scopeRotation = ref(0.0);
const scopeSize = ref(0.5);
const scopeOffset = ref([0.0, 0.0]);
const scopeOffsetVel = ref([0.0, 0.0]);
const scopeSizeVel = ref(0.0);
const scopeRotationVel = ref(0.0);
const isUserPressing = ref(false);
const keyPressedW = ref(false);
const keyPressedA = ref(false);
const keyPressedS = ref(false);
const keyPressedD = ref(false);
const keyPressedI = ref(false);
const keyPressedJ = ref(false);
const keyPressedK = ref(false);
const keyPressedL = ref(false);
const saveNextFrame = ref(false);
const canvas = useTemplateRef('canvas');
const saveButton = useTemplateRef('saveButton');
const saveButtonVisible = ref(false);

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
    console.info('Failed to get environment camera. Trying any camera, under the assumption it is a user-facing camera', e);
    try {
      stream = await navigator.mediaDevices.getUserMedia({video: true, audio: false});
      facingMode.value = stream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
    } catch (e2) {
      if ((e2 as Error).name === 'ConstraintNotSatisfiedError') {
        console.error('Device has no camera', e2);
      } else if ((e2 as Error).name === 'PermissionDeniedError') {
        console.error('Permissions not accepted', e2);
      } else {
        console.error('Other error', e2);
      }
    }
  }

  if (stream !== null) {
    camera.srcObject = stream;
    camera.play();
  }

  // Canvas with WebGL context
  const canvas = document.getElementById('maincanvas') as HTMLCanvasElement;
  const canvasSize = Math.max(1024, window.innerWidth, window.innerHeight) * window.devicePixelRatio;
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
      precision highp float;

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
          if (scopeShape == ${ScopeShape.Equilateral}) {
            k = equilateral(k, scopeSize, scopeRotation, d);
            scopeDiameterRatio = 1.0;
          } else if (scopeShape == ${ScopeShape.Square}) {
            k = square(k, scopeSize, scopeRotation, d);
            scopeDiameterRatio = sqrt(2.0);
          } else if (scopeShape == ${ScopeShape.Isosceles}) {
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
    // Handle keyboard state
    if (keyPressedA.value && keyPressedD.value) {
      // Do nothing
    } else if (keyPressedA.value) {
      scopeRotationVel.value -= 0.0002;
    } else if (keyPressedD.value) {
      scopeRotationVel.value += 0.0002;
    }
    if (keyPressedW.value && keyPressedS.value) {
      // Do nothing
    } else if (keyPressedW.value) {
      scopeSizeVel.value += 0.0002;
    } else if (keyPressedS.value) {
      scopeSizeVel.value -= 0.0002;
    }
    if (keyPressedI.value && keyPressedK.value) {
      // Do nothing
    } else if (keyPressedI.value) {
      scopeOffsetVel.value[0] -= 0.0002 / scopeSize.value;
    } else if (keyPressedK.value) {
      scopeOffsetVel.value[0] += 0.0002 / scopeSize.value;
    }
    if (keyPressedJ.value && keyPressedL.value) {
      // Do nothing
    } else if (keyPressedJ.value) {
      scopeOffsetVel.value[1] += 0.0002 / scopeSize.value;
    } else if (keyPressedL.value) {
      scopeOffsetVel.value[1] -= 0.0002 / scopeSize.value;
    }

    let scopeRotationOffset = 0;
    if (props.scopeShape === ScopeShape.Equilateral) {
      scopeRotationOffset = Math.PI / 3;
    } else if (props.scopeShape === ScopeShape.Isosceles) {
      scopeRotationOffset = -Math.PI / 2;
    } else if (props.scopeShape === ScopeShape.Scalene) {
      scopeRotationOffset = -Math.PI / 2;
    }

    scopeOffset.value[0] += Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] - Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];
    scopeOffset.value[1] += Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] + Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];

    if (props.scopeAutoRotationVelocity !== 0) {
      scopeRotationVel.value = props.scopeAutoRotationVelocity / 25;
      scopeRotation.value += scopeRotationVel.value;
    }
    if (touchOrigin1 === null && mousePrevPosition === null) {
      scopeRotation.value += scopeRotationVel.value;
      scopeRotationVel.value *= 0.99;
      scopeSizeVel.value *= 0.95;
      scopeSize.value *= 1 + Math.min(scopeSizeVel.value, 0.99);
      if (scopeSize.value > 2.0) {
        scopeSize.value -= (scopeSize.value - 2) / 10;
      }
      if (scopeSize.value < Math.pow(0.5, 7)) {
        scopeSize.value += (Math.pow(0.5, 7) - scopeSize.value) / 10;
      }
    }
    scopeOffsetVel.value[0] *= 0.95;
    scopeOffsetVel.value[1] *= 0.95;

    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, camera);
    gl.uniform2f(dataDimensionsBind, camera.videoWidth, camera.videoHeight);
    gl.uniform1i(dataIsFacingUserBind, facingMode.value === 'user' ? 1 : 0);
    gl.uniform1i(scopeShapeBind, props.scopeShape);
    gl.uniform1f(scopeRotationBind, scopeRotation.value + scopeRotationOffset);
    gl.uniform1f(scopeSizeBind, scopeSize.value);
    gl.uniform2f(scopeOffsetBind, scopeOffset.value[0], scopeOffset.value[1]);
    gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);
    gl.drawArrays(gl.TRIANGLES, 0, 6);

    if (saveNextFrame.value) {
      const image = canvas
          .toDataURL('image/jpeg', 1.0);
      const a = document.createElement('a');
      a.href = image;
      a.download = 'kaleidoscope.jpg';
      a.click();
      saveNextFrame.value = false;
    }

    requestAnimationFrame(animate);
  }
  animate();
}

interface Point {
  clientX: number;
  clientY: number;
}
let mousePrevPosition = null as null|{x: number, y: number};
let touchId1: null|number = null;
let touchOrigin1: null|Point = null;
let touchPrevTime = new Date().getTime();
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
  touchOrigin1 = touch;
  touchPrevTime = new Date().getTime();
  isUserPressing.value = true;
  scopeRotationVel.value = 0;
  scopeSizeVel.value = 0;
}

function touchMoveCallback(event: TouchEvent) {
  if (touchId1 === null || touchPrev1 === null || touchOrigin1 === null) {
    return;
  }
  const touch = getTouchById(event.changedTouches, touchId1);
  if (touch === null) {
    return;
  }
  if (new Date().getTime() - touchPrevTime < 0.001) {
    return;
  }
  const deltaTime = Math.max(new Date().getTime() - touchPrevTime, 0.001);
  const deltaX = (touch.clientX - touchPrev1.clientX) / deltaTime;
  const deltaY = (touch.clientY - touchPrev1.clientY) / deltaTime;

  scopeRotation.value += deltaX / 10;
  if (Math.abs(touch.clientX - touchPrev1.clientX) > 1) {
    scopeRotationVel.value = deltaX / 10;
  } else {
    scopeRotationVel.value = 0;
  }

  scopeSize.value *= 1.0 + deltaY / 10;
  if (Math.abs(touch.clientY - touchPrev1.clientY) > 1) {
    scopeSizeVel.value = deltaY / 10;
  } else {
    scopeSizeVel.value = 0;
  }

  touchPrevTime = new Date().getTime();
  touchPrev1 = touch;
}

function touchEndCallback(event: TouchEvent) {
  if (touchId1 === null || touchPrev1 === null || touchOrigin1 === null) {
    return;
  }
  const touch = getTouchById(event.changedTouches, touchId1);
  if (touch === null) {
    return;
  }
  isUserPressing.value = false;
  touchId1 = null;
  touchPrev1 = null;
  touchOrigin1 = null;
}

function touchCancelCallback() {
  isUserPressing.value = false;
  touchId1 = null;
  touchPrev1 = null;
  touchOrigin1 = null;
}

onMounted(() => {
  main();

  const canvasElement = canvas.value as HTMLCanvasElement;

  canvasElement.addEventListener('mousedown', (mouseEvent) => {
    // Left mouse button only
    if (mouseEvent.button !== 0) {
      return;
    }
    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
    isUserPressing.value = true;
    scopeRotationVel.value = 0;
    scopeSizeVel.value = 0;
  });
  document.addEventListener('mousemove', (mouseEvent) => {
    if (mousePrevPosition === null) {
      return;
    }
    const deltaX = (mouseEvent.clientX - mousePrevPosition.x) / 10;
    const deltaY = (mouseEvent.clientY - mousePrevPosition.y) / 10;

    scopeRotation.value += deltaX / 20;
    if (Math.abs(mouseEvent.clientX - mousePrevPosition.x) > 1) {
      scopeRotationVel.value = deltaX / 20;
    } else {
      scopeRotationVel.value = 0;
    }

    scopeSize.value *= 1.0 + deltaY / 50;
    if (Math.abs(mouseEvent.clientY - mousePrevPosition.y) > 1) {
      scopeSizeVel.value = deltaY / 50;
    } else {
      scopeSizeVel.value = 0;
    }

    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
  });
  document.addEventListener('mouseup', () => {
    if (mousePrevPosition === null) {
      return;
    }

    isUserPressing.value = false;
    mousePrevPosition = null;
  });

  document.addEventListener('wheel', (wheelEvent) => {
    wheelEvent.preventDefault();
    scopeRotation.value += wheelEvent.deltaX / 500;
    scopeSize.value *= 1.0 - wheelEvent.deltaY / 500;
  });

  canvasElement.addEventListener('touchstart', touchStartCallback);
  canvasElement.addEventListener('touchmove', touchMoveCallback);
  canvasElement.addEventListener('touchend', touchEndCallback);
  canvasElement.addEventListener('touchcancel', touchCancelCallback);

  document.addEventListener('keydown', (keyEvent) => {
    if (keyEvent.code == 'KeyW') {
      keyPressedW.value = true;
    } else if (keyEvent.code == 'KeyA') {
      keyPressedA.value = true;
    } else if (keyEvent.code == 'KeyS') {
      keyPressedS.value = true;
    } else if (keyEvent.code == 'KeyD') {
      keyPressedD.value = true;
    } else if (keyEvent.code == 'KeyI') {
      keyPressedI.value = true;
    } else if (keyEvent.code == 'KeyJ') {
      keyPressedJ.value = true;
    } else if (keyEvent.code == 'KeyK') {
      keyPressedK.value = true;
    } else if (keyEvent.code == 'KeyL') {
      keyPressedL.value = true;
    }
  });
  document.addEventListener('keyup', (keyEvent) => {
    if (keyEvent.code == 'KeyW') {
      keyPressedW.value = false;
    } else if (keyEvent.code == 'KeyA') {
      keyPressedA.value = false;
    } else if (keyEvent.code == 'KeyS') {
      keyPressedS.value = false;
    } else if (keyEvent.code == 'KeyD') {
      keyPressedD.value = false;
    } else if (keyEvent.code == 'KeyI') {
      keyPressedI.value = false;
    } else if (keyEvent.code == 'KeyJ') {
      keyPressedJ.value = false;
    } else if (keyEvent.code == 'KeyK') {
      keyPressedK.value = false;
    } else if (keyEvent.code == 'KeyL') {
      keyPressedL.value = false;
    }
  });

  function displayContextMenu (at: Point, touchscreen: boolean) {
    const saveButtonElement = saveButton.value as HTMLButtonElement;
    if (!touchscreen) {
      saveButtonElement.style.left = 'calc(min(100dvw - 7.5rem,' + at.clientX + 'px + 3px))';
      saveButtonElement.style.top = 'calc(min(100dvh - 7rem,' + at.clientY + 'px - 3px))';
    } else {
      saveButtonElement.style.left = 'calc(max(0rem, min(100dvw - 7.5rem,' + at.clientX + 'px - 7.5rem / 2.0)))';
      saveButtonElement.style.top = 'calc(max(0rem, min(100dvh - 7rem,' + at.clientY + 'px - 6rem)))';
    }
    saveButtonVisible.value = true;
  }

  const touchContextMenuDuration = 1000;
  let touchContextMenuId = null as null | number;
  let touchContextMenuPrev = null as null | Point;
  let touchContextMenuDistance = 0;
  let touchContextMenuTimeout = null as null | number;
  canvasElement.addEventListener('touchstart', (touchEvent) => {
    saveButtonVisible.value = false;
    const touch = touchEvent.changedTouches[0];
    touchContextMenuId = touch.identifier;
    touchContextMenuDistance = 0;
    const originalTouchContextMenuId = touchContextMenuId;
    touchContextMenuPrev = touch;

    touchContextMenuTimeout = setTimeout(() => {
      console.log(touchContextMenuId, originalTouchContextMenuId);
      console.log(touchContextMenuDistance);
      if (touchContextMenuId !== originalTouchContextMenuId) {
        return;
      }
      if (touchContextMenuDistance > 2) {
        return;
      }
      displayContextMenu(touch, true);
    }, touchContextMenuDuration);
  });
  document.addEventListener('touchmove', (touchEvent) => {
    if (touchContextMenuId === null || touchContextMenuPrev == null) {
      return;
    }
    const touch = getTouchById(touchEvent.changedTouches, touchContextMenuId);
    if (touch === null) {
      return;
    }
    const dx = touch.clientX - touchContextMenuPrev.clientX;
    const dy = touch.clientY - touchContextMenuPrev.clientY;
    const movedDistance = Math.sqrt(dx*dx + dy*dy);
    touchContextMenuDistance += movedDistance;
    touchContextMenuPrev = touch;
  });
  document.addEventListener('touchend', (touchEvent) => {
    if (touchContextMenuId === null) {
      return;
    }
    const touch = getTouchById(touchEvent.changedTouches, touchContextMenuId);
    if (touch === null) {
      return;
    }
    touchContextMenuId = null;
    touchContextMenuPrev = null;
    touchContextMenuDistance = 0;
    if (touchContextMenuTimeout !== null) {
      clearTimeout(touchContextMenuTimeout);
    }
  });


  canvasElement.addEventListener('contextmenu', function(e) {
    displayContextMenu(e, false);
    e.preventDefault();
  });
  document.addEventListener('mousedown', () => {
    saveButtonVisible.value = false;
  });
  window.addEventListener('resize', () => {
    saveButtonVisible.value = false;
  });
});

</script>

<template>
  <canvas
    id="maincanvas"
    ref="canvas"
    class="bg-black"
    style="width:100dvw;height:100dvh;object-fit:cover"
  />
  <div
    class="pointer-events-none absolute left-0 top-0 p-5 pb-24 overflow-hidden"
    style="width:100dvw;height:100dvh"
  >
    <div>
      <button
        ref="saveButton"
        class="
          absolute
          flex-nowrap
          whitespace-nowrap
          text-nowrap
          flex
          flex-row
          items-center
          gap-1
          bg-neutral-200
          bg-opacity-70
          hover:bg-opacity-80
          dark:bg-neutral-800
          dark:bg-opacity-70
          shadow-xl
          dark:shadow-none
          backdrop-blur-xl
          rounded-md
          p-1
          px-2
          pointer-events-auto
          cursor-pointer
          text-black
          dark:text-white
        "
        :class="{
          'hidden': !saveButtonVisible
        }"
        style="left: 100%;top:100%"
        @click="saveNextFrame = true; saveButtonVisible = false"
        @mousedown.stop
      >
        <picture>
          <source
            srcset="/download-dark.svg"
            media="(prefers-color-scheme: dark)"
          >
          <img
            src="/download-light.svg"
            alt="Download icon"
            class="h-4 min-w-4"
          >
        </picture>
        Save image
      </button>
    </div>
  </div>
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
