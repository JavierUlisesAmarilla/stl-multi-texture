<script setup lang="ts">


import * as THREE from "three"
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { onMounted } from "vue"


let firstMounted = true


onMounted(() => {
  if (!firstMounted) {
    return
  }
  firstMounted = false

  const scene = new THREE.Scene()
  const camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 1, 8000)
  camera.position.z = 500
  camera.lookAt(0, 0, 0)

  const renderer = new THREE.WebGLRenderer({
    canvas: document.querySelector("#bg") as HTMLCanvasElement,
    antialias: true,
    alpha: true
  })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setClearColor(0x000000, 0.9)

  const axesHelper = new THREE.AxesHelper(1000)

  scene.add(axesHelper)

  const controls = new OrbitControls(camera, renderer.domElement)

  const vertexShader = `
    attribute vec3 aPosition;
    attribute vec3 aColor;

    varying vec3 vColor;
    varying vec3 vNormal;

    void main(){
      vColor = aColor;
      vNormal = normal;
      vec3 transformed = position.xyz;
      transformed += aPosition;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(transformed, 1.);
    }
  `
  const fragmentShader = `
    uniform vec3 lightDirection;
    
    varying vec3 vColor;
    varying vec3 vNormal;

    vec3 lightColor = vec3(1.0, 1.0, 1.0);

    void main(){
      vec3 norm = normalize(vNormal);
      float nDotL = clamp(dot(lightDirection, norm), 0.0, 1.0);
      vec3 diffuseColor = lightColor * nDotL * vColor;
      gl_FragColor = vec4(diffuseColor, 1.);
    }
  `

  const material = new THREE.ShaderMaterial({
    uniforms: {
      lightDirection: { value: new THREE.Vector3(1.0, 1.0, 1.0).normalize() }
    },
    fragmentShader: fragmentShader,
    vertexShader: vertexShader
  })

  const baseGeom = new THREE.SphereGeometry(20, 32, 16)
  const bufferGeom = new THREE.BufferGeometry().fromGeometry(baseGeom)
  const colorArr: any = []
  const posArr = []
  new THREE.Color(Math.random(), Math.random(), Math.random()).toArray(colorArr, 0)
  posArr.push(Math.random() * 400 - 200, Math.random() * 400 - 200, Math.random() * 400 - 200)
  bufferGeom.setAttribute('aColor', new THREE.InstancedBufferAttribute(new Float32Array(colorArr), 3))
  bufferGeom.setAttribute('aPosition', new THREE.InstancedBufferAttribute(new Float32Array(posArr), 3))

  const bufferMesh = new THREE.Mesh(bufferGeom, material)
  // bufferMesh.position.set(100, 100, 100)

  scene.add(bufferMesh)

  animate()

  function animate() {
    requestAnimationFrame(animate)
    renderer.render(scene, camera)
    controls.update()
  }
})


</script>

<template>
  <div>
    <canvas id="bg" style="width: 100vw; height: 100vh; position: fixed; top: 0; left: 0;"></canvas>
  </div>
</template>
