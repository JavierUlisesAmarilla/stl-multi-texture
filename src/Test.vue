<script setup lang="ts">

import * as THREE from "three";
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { onMounted } from "vue";

const scene = new THREE.Scene();
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let firstMounted = true

onMounted(() => {
  if (!firstMounted) {
    return
  }
  firstMounted = false

  const width = 800
  const height = 800

  camera = new THREE.PerspectiveCamera(75, height / width, 0.1, 1000);

  camera.lookAt(0, 0, 0);
  camera.position.z = 150;
  camera.position.y = 200;

  renderer = new THREE.WebGLRenderer({
    canvas: document.querySelector("#bg") as HTMLCanvasElement,
    alpha: true
  });

  controls = new OrbitControls(camera, renderer.domElement)

  const light = new THREE.PointLight(0xFFFFFF, 0.8)
  light.position.set(0, 100, 200)
  scene.add(light)

  const color = 0xFFFFFF;
  const intensity = 1;
  const light2 = new THREE.AmbientLight(color, intensity);
  scene.add(light2);

  renderer.setSize(width, height);
  renderer.render(scene, camera);

  /* Start Test */

  const textureLoader = new THREE.TextureLoader()
  const texture1 = textureLoader.load('/textures/cherry.jpg')
  const texture2 = textureLoader.load('/textures/man.jpg')

  const shaderMat = new THREE.ShaderMaterial({
    uniforms: {
      texture1: { value: texture1 },
      texture2: { value: texture2 },
    },
    vertexShader: `
      precision highp float;
      precision highp int;
      varying vec2 vUv;

      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      precision mediump float;
      uniform sampler2D texture1;
      uniform sampler2D texture2;
      varying vec2 vUv;

      void main() {
        vec4 col1 = texture2D(texture1, vUv);
        vec4 col2 = texture2D(texture2, vUv);
        col2 = col2.a > 0.5 ? col2 : vec4(0, 0, 0, 1);
        gl_FragColor = mix( col1, col2, 0.25 );
      }
    `
  });

  const sphereMesh = new THREE.Mesh(
    new THREE.BoxGeometry(100, 100, 100, 100, 100, 100),
    shaderMat,
  )

  scene.add(sphereMesh)

  /* End Test */

  animate();
});

function animate() {
  controls.update()
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

</script>

<template>
  <div>
    <canvas id="bg" class="" width="800" height="800"></canvas>
  </div>
</template>
