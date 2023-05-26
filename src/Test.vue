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

  getSvgTexture('/textures/overlay.svg').then((texture1) => {
    texture1.needsUpdate = true
    const textureLoader = new THREE.TextureLoader()
    const texture2 = textureLoader.load('/textures/cherry.jpg')
    // texture1 = textureLoader.load('/textures/man.jpg')

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
          gl_FragColor = mix( col1, col2, 0.5 );
        }
      `
    });

    const sphereMesh = new THREE.Mesh(
      new THREE.BoxGeometry(100, 100, 100, 100, 100, 100),
      shaderMat,
    )

    scene.add(sphereMesh)
  })

  /* End Test */

  animate();
});

function animate() {
  controls.update()
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

async function getSvgTexture(svgUrl: string) {
  const fileLoader = new THREE.FileLoader()
  const imageLoader = new THREE.ImageLoader()
  const svgStr = await fileLoader.loadAsync(svgUrl)
  const parser = new DOMParser()
  const svgEl: any = parser.parseFromString(svgStr, 'image/svg+xml').documentElement
  const newSvgData = (new XMLSerializer()).serializeToString(svgEl)
  const dataUrl = `data:image/svg+xml;base64,${window.btoa(unescape(encodeURIComponent(newSvgData)))}`
  const imageTag = await imageLoader.loadAsync(dataUrl)
  const svgCanvas = document.createElement('canvas')
  svgCanvas.width = svgEl.width?.baseVal?.value
  svgCanvas.height = svgEl.height?.baseVal?.value
  const ctx = svgCanvas.getContext('2d')
  ctx?.drawImage(imageTag, 0, 0)
  const svgTexture = new THREE.Texture(svgCanvas)
  return svgTexture
}

</script>

<template>
  <div>
    <canvas id="bg" class="" width="800" height="800"></canvas>
  </div>
</template>
