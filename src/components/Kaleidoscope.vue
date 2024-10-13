<script setup lang="ts">
import { onMounted } from 'vue';

async function main() {
  // Capture webcam input using invisible `video` element
  // Adapted from p5js.org/examples/3d-shader-using-webcam.html
  const camera = document.getElementById('camera') as HTMLVideoElement;

  // Ask user permission to record their camera
  const stream = await navigator.mediaDevices.getUserMedia({video: { facingMode: 'environment' }, audio: false});
  camera.srcObject = stream;
  camera.play();

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
      uniform vec2 canvasDimensions;

      float round(float v) {
          return floor(v + 0.5);
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

      vec2 square_float_to_axial_hex_grid(vec2 pos) {
        float SQRT3 = sqrt(3.0);
        float ratio = 2.0 / 3.0;
        pos.y = ratio * 0.5 * (SQRT3*pos.y - pos.x);
        pos.x *= ratio;
        return axial_round(pos);
      }

      vec2 hexToCentroid(vec2 hex) {
        vec2 pos = vec2(0.0,0.0);
        float size = 2.0 / 3.0;
        pos.x = hex.x / size;
        pos.y = hex.y * 2.0 / sqrt(3.0) / size + hex.x / sqrt(3.0) / size;
        return pos;
      }

      vec2 square(vec2 u, float kLength) {
        float kOffset = (1.0/kLength - 1.0) / (1.0/kLength * 2.0);
        vec2 k = vec2(0.0, 0.0);
        if (mod(-kOffset + u.x, kLength * 2.0) > kLength) {
          k.x = 1.0 - mod(-kOffset + u.x, kLength) / kLength;
        } else {
          k.x = mod(-kOffset + u.x, kLength) / kLength;
        }
        if (mod(-kOffset + u.y, kLength * 2.0) > kLength) {
          k.y = 1.0 - mod(-kOffset + u.y, kLength) / kLength;
        } else {
          k.y = mod(-kOffset + u.y, kLength) / kLength;
        }
        return k;
      }

      vec2 equilateral(vec2 u, float kLength) {
        u.x -= (1.0/kLength - 1.0) / (1.0/kLength * 2.0);
        u.y -= (1.0/kLength - (sqrt(3.0) / 2.0)) / (1.0/kLength * 2.0) / (sqrt(3.0) / 2.0);

        u /= kLength;

        vec2 k = vec2(0.0, 0.0);

        vec2 hexIndex = square_float_to_axial_hex_grid(u);
        vec2 hexCentroid = hexToCentroid(hexIndex);
        // For debugging
        // k = vec2(distance(hexToCentroid(hexIndex), u));
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

        return k;
      }

      void main() {
          // The interesting things to change!
          float kLength = 1.0 / 10.0; // Effectively, how often should the image reflect 1 is one reflection (when viewed in square mode)
          float dataScopePercentage = 0.5; // Effectively how narrow the kaleidoscope should be, as a percentage of the camera size

          vec2 centerOffset = vec2(0.0);

          vec2 fragCoord = vec2(
            gl_FragCoord.x / canvasDimensions.x,
            gl_FragCoord.y / canvasDimensions.y
          );

          // Position in kaleidoscope space
          // In opengl, y-coordinate is flipped
          vec2 u = vec2(fragCoord.x,1.0-fragCoord.y);

          // Square kaleidoscope mode
          // vec2 k = square(u, kLength);

          // Equilateral triangle mode
          vec2 k = equilateral(u, kLength);

          // Now map the k value to coordinates on the image
          // 0,0 will be the centre of the image
          // 1,1 will be the top right of the image (not the bottom left– It's easier to orientate if things are up-right)
          float dataMinDimension = min(dataDimensions.x, dataDimensions.y);
          float dataWindowSize = dataMinDimension / 2.0 * dataScopePercentage;
          vec2 i = vec2(0.0,0.0);
          // Todo: Account for front-facing cameras, and flip the x-coordinate, because people understand mirrors more easily.
          i.x = dataDimensions.x / 2.0 + (-dataWindowSize / 2.0 + k.x * dataWindowSize);
          i.y = dataDimensions.y / 2.0 - (-dataWindowSize / 2.0 + k.y * dataWindowSize); // y-axis is flipped because of openGL coordinate space

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
  const canvasDimensionsBind = gl.getUniformLocation(program, 'canvasDimensions');

  // Repeatedly pull camera data and render
  function animate(){
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, camera);
    gl.uniform2f(dataDimensionsBind, camera.videoWidth, camera.videoHeight);
    gl.uniform2f(dataDimensionsBind, 640, 420);
    gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);
    gl.drawArrays(gl.TRIANGLES, 0, 6);
    requestAnimationFrame(animate);
  }
  animate();
}

onMounted(() => {
  main();
});
</script>

<template>
  <canvas
    id="maincanvas"
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
