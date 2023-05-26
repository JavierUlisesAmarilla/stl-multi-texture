<script setup lang="ts">


import * as THREE from "three"
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'
// import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader'
// import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { onMounted } from "vue"


const scene = new THREE.Scene()
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let orbitControls: OrbitControls
let firstMounted = true


onMounted(() => {
  if (!firstMounted) {
    return
  }
  firstMounted = false

  // Scene
  scene.background = new THREE.Color(0x666666)
  scene.fog = new THREE.FogExp2(0x001f3f, 0.002)

  // Lights
  const sphere = new THREE.SphereGeometry(0.1, 16, 16)

  const light1 = new THREE.PointLight(0xff0040, 20, 50)
  light1.add(new THREE.Mesh(sphere, new THREE.MeshBasicMaterial({ color: 0xff0040 })))
  light1.position.set(0, 3, 0)
  scene.add(light1)

  const light2 = new THREE.PointLight(0x0040ff, 2, 50);
  light2.add(new THREE.Mesh(sphere, new THREE.MeshBasicMaterial({ color: 0x0040ff })));
  light2.position.set(3, 0, 0)
  scene.add(light2);

  const light3 = new THREE.PointLight(0x80ff80, 2, 50);
  light3.add(new THREE.Mesh(sphere, new THREE.MeshBasicMaterial({ color: 0x80ff80 })));
  light3.position.set(0, 0, 3)
  scene.add(light3);

  const light4 = new THREE.PointLight(0xffaa00, 2, 50);
  light4.add(new THREE.Mesh(sphere, new THREE.MeshBasicMaterial({ color: 0xffaa00 })));
  light4.position.set(-3, 0, 0)
  scene.add(light4);

  // Camera
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 6, 6)

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: document.querySelector("#bg") as HTMLCanvasElement,
    antialias: true,
    alpha: true,
  })

  // Orbit Controls
  orbitControls = new OrbitControls(camera, renderer.domElement)

  // Helper
  scene.add(new THREE.AxesHelper(10))

  // Init
  resize()
  animate()

  getModelMesh('models/model.stl', 'textures/yellow.png', 'textures/overlay.svg').then((modelMesh) => {
    modelMesh.rotation.x = -Math.PI / 2
    modelMesh.position.set(-6, 0, 3)
    scene.add(modelMesh)
  })

  // const objLoader = new OBJLoader()
  // objLoader.load('models/walt/WaltHead.obj', function (obj) {
  //   obj.scale.multiplyScalar(0.05)
  //   obj.position.setY(-1)
  //   scene.add(obj)
  // })

  // const gltfLoader = new GLTFLoader()
  // gltfLoader.load('models/Xbot.glb', function (gltf) {
  //   gltf.scene.scale.multiplyScalar(2)
  //   gltf.scene.position.setY(-1)
  //   scene.add(gltf.scene)
  // });
})


function animate() {
  orbitControls.update()
  renderer.render(scene, camera)
  requestAnimationFrame(animate)
}


function resize() {
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
}


window.addEventListener('resize', resize)


async function getModelMesh(modelPath: string, texturePath: string, svgPath: string) {
  const stlLoader = new STLLoader()
  const textureLoader = new THREE.TextureLoader()
  const stlGeo = await stlLoader.loadAsync(modelPath)
  const texture1 = await getSvgTexture(svgPath)
  texture1.needsUpdate = true
  const texture2 = textureLoader.load(texturePath)

  const stlMaterial = new THREE.ShaderMaterial({
    uniforms: {
      ...THREE.UniformsLib.lights,
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
    `,
    lights: true,
    blending: THREE.NoBlending,
  })

  console.log('stlMaterial: ', stlMaterial)

  const modelMesh = getStlMesh(stlGeo, stlMaterial)
  return modelMesh
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


function getStlMesh(geometry: any, material: any) {
  let rawGeometry = geometry
  if (geometry.isBufferGeometry) {
    rawGeometry = new THREE.Geometry().fromBufferGeometry(geometry)
  }
  rawGeometry.computeBoundingBox()
  const max = rawGeometry.boundingBox.max,
    min = rawGeometry.boundingBox.min
  const offset = new THREE.Vector2(0 - min.x, 0 - min.y)
  const range = new THREE.Vector2(max.x - min.x, max.y - min.y)
  const faces = rawGeometry.faces
  rawGeometry.faceVertexUvs[0] = []

  for (let i = 0; i < faces.length; i++) {
    const v1 = rawGeometry.vertices[faces[i].a],
      v2 = rawGeometry.vertices[faces[i].b],
      v3 = rawGeometry.vertices[faces[i].c]
    rawGeometry.faceVertexUvs[0].push([
      new THREE.Vector2((v1.x + offset.x) / range.x, (v1.y + offset.y) / range.y),
      new THREE.Vector2((v2.x + offset.x) / range.x, (v2.y + offset.y) / range.y),
      new THREE.Vector2((v3.x + offset.x) / range.x, (v3.y + offset.y) / range.y)
    ])
  }

  rawGeometry.uvsNeedUpdate = true

  const stlMesh = new THREE.Mesh(rawGeometry, material)
  return stlMesh
}


</script>

<template>
  <div>
    <canvas id="bg" style="width: 100vw; height: 100vh; position: fixed; top: 0; left: 0;"></canvas>
  </div>
</template>
