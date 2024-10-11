<script setup lang="ts">
import { onMounted } from 'vue';

async function main() {
  // Capture webcam input using invisible `video` element
  // Adapted from p5js.org/examples/3d-shader-using-webcam.html
  const camera = document.getElementById('camera') as HTMLVideoElement;

  // Ask user permission to record their camera
  const stream = await navigator.mediaDevices.getUserMedia({video: true, audio: false});
  camera.srcObject = stream;
  camera.play();

  // 512×512 Canvas with WebGL context
  const canvas = document.getElementById('maincanvas') as HTMLCanvasElement;
  const gl = canvas.getContext('webgl')!;
  canvas.width = canvas.height = 512;
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
      uniform sampler2D data;
      void main() {

          if (mod(gl_FragCoord.x, 200.0) > 100.0) {
            if (mod(gl_FragCoord.y, 200.0) > 100.0) {
            gl_FragColor=texture2D(data,vec2(mod(-gl_FragCoord.x, 100.0), mod(gl_FragCoord.y, 100.0))/vec2(512,512)).xyzw;
          } else {
            gl_FragColor=texture2D(data,vec2(mod(gl_FragCoord.x, 100.0), mod(-gl_FragCoord.y, 100.0))/vec2(512,512)).xyzw;
          }
          } else {
            if (mod(gl_FragCoord.y, 200.0) > 100.0) {
            gl_FragColor=texture2D(data,vec2(mod(-gl_FragCoord.x, 100.0), mod(gl_FragCoord.y, 100.0))/vec2(512,512)).xyzw;
          } else {
            gl_FragColor=texture2D(data,vec2(mod(gl_FragCoord.x, 100.0), mod(-gl_FragCoord.y, 100.0))/vec2(512,512)).xyzw;
          }
          }

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
  const param = gl.getActiveUniform(program,0); // data bind point
  gl.uniform1i(gl.getUniformLocation(program,'data'),0);
  gl.activeTexture(gl.TEXTURE0);
  gl.bindTexture(gl.TEXTURE_2D,texture);

  // Repeatedly pull camera data and render
  function animate(){
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, camera);
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
    style="width:512px;height:512px;"
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
