<script setup lang="ts">


import * as THREE from "three"
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'
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

  // Camera
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(-9, 3, 6)

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: document.querySelector("#bg") as HTMLCanvasElement,
    antialias: true,
    alpha: true,
  })

  // Orbit Controls
  orbitControls = new OrbitControls(camera, renderer.domElement)

  // Init
  resize()
  animate()

  getModelMesh(
    'models/model.stl',
    'textures/overlay.svg',
    'textures/top.jpg',
    'textures/front.jpg',
    'textures/back.jpg',
    'textures/left.jpg',
    'textures/right.jpg',
    'textures/bottom.jpg',
  ).then((modelMesh) => {
    modelMesh.rotation.x = -Math.PI / 2
    modelMesh.position.set(-6, 0, 3)
    // modelMesh.add(new THREE.AxesHelper(10))
    scene.add(modelMesh)
  })
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


async function getModelMesh(
  modelPath: string,
  svgPath: string,
  topTexturePath: string,
  frontTexturePath: string,
  backTexturePath: string,
  leftTexturePath: string,
  rightTexturePath: string,
  bottomTexturePath: string,
) {
  const stlLoader = new STLLoader()
  const textureLoader = new THREE.TextureLoader()
  const stlGeo = await stlLoader.loadAsync(modelPath)
  const topTexture1 = await getSvgTexture(svgPath)
  topTexture1.needsUpdate = true
  const topTexture2 = textureLoader.load(topTexturePath)
  const frontTexture = textureLoader.load(frontTexturePath)
  const backTexture = textureLoader.load(backTexturePath)
  const leftTexture = textureLoader.load(leftTexturePath)
  const rightTexture = textureLoader.load(rightTexturePath)
  const bottomTexture = textureLoader.load(bottomTexturePath)

  const stlMaterial = new THREE.ShaderMaterial({
    uniforms: {
      lightDirection: { value: new THREE.Vector3(-1, -1, 1).normalize() },
      lightColor: { value: new THREE.Vector4(1, 1, 1, 1) },
      topTexture1: { value: topTexture1 },
      topTexture2: { value: topTexture2 },
      frontTexture: { value: frontTexture },
      backTexture: { value: backTexture },
      leftTexture: { value: leftTexture },
      rightTexture: { value: rightTexture },
      bottomTexture: { value: bottomTexture },
      min: { value: new THREE.Vector3() },
      max: { value: new THREE.Vector3() },
      tolerance: { value: 0.001 },
    },
    vertexShader: `
      varying vec2 vUv;
      varying vec3 vNormal;
      varying vec3 vPosition;

      void main() {
        vUv = uv;
        vNormal = normal;
        vPosition = position;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1);
      }
    `,
    fragmentShader: `
      uniform vec3 lightDirection;
      uniform vec4 lightColor;
      uniform sampler2D topTexture1;
      uniform sampler2D topTexture2;
      uniform sampler2D frontTexture;
      uniform sampler2D backTexture;
      uniform sampler2D leftTexture;
      uniform sampler2D rightTexture;
      uniform sampler2D bottomTexture;
      uniform vec3 min;
      uniform vec3 max;
      uniform float tolerance;

      varying vec2 vUv;
      varying vec3 vNormal;
      varying vec3 vPosition;

      void main() {
        vec4 mixColor = vec4(0);

        float minMinX = min.x - tolerance;
        float maxMinX = min.x + tolerance;

        float minMaxX = max.x - tolerance;
        float maxMaxX = max.x + tolerance;

        float minMinY = min.y - tolerance;
        float maxMinY = min.y + tolerance;

        float minMaxY = max.y - tolerance;
        float maxMaxY = max.y + tolerance;

        float minMinZ = min.z - tolerance;
        float maxMinZ = min.z + tolerance;

        float minMaxZ = max.z - tolerance;
        float maxMaxZ = max.z + tolerance;

        if (vPosition.y >= minMinY && vPosition.y <= maxMinY) {
          mixColor = texture2D(frontTexture, vUv);
        } else if (vPosition.y >= minMaxY && vPosition.y <= maxMaxY) {
          mixColor = texture2D(backTexture, vUv);
        } else if (vPosition.x >= minMinX && vPosition.x <= maxMinX) {
          mixColor = texture2D(leftTexture, vUv);
        } else if (vPosition.x >= minMaxX && vPosition.x <= maxMaxX) {
          mixColor = texture2D(rightTexture, vUv);
        } else if (vPosition.z >= minMinZ && vPosition.z <= maxMinZ) {
          mixColor = texture2D(bottomTexture, vUv);
        } else {
          vec4 topColor1 = texture2D(topTexture1, vUv);
          vec4 topColor2 = texture2D(topTexture2, vUv);
          topColor2 = topColor2.a > .5 ? topColor2 : vec4(0, 0, 0, 1);
          mixColor = mix(topColor1, topColor2, .5);
          vec3 norm = normalize(vNormal);
          float nDotL = clamp(dot(lightDirection, norm), 0., 1.);
          mixColor = lightColor * mixColor * nDotL;
        }

        gl_FragColor = mixColor;
      }
    `,
    // wireframe: true,
  })

  // const stlMesh = new THREE.Mesh(stlGeo, stlMaterial)
  const { stlMesh, min, max } = getStlMesh(stlGeo, stlMaterial)
  stlMaterial.uniforms.min.value = min
  stlMaterial.uniforms.max.value = max
  return stlMesh
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
  rawGeometry.computeVertexNormals()
  const max = rawGeometry.boundingBox.max,
    min = rawGeometry.boundingBox.min
  const offset = new THREE.Vector3(0 - min.x, 0 - min.y, 0 - min.z)
  const range = new THREE.Vector3(max.x - min.x, max.y - min.y, max.z - min.z)
  const faces = rawGeometry.faces
  rawGeometry.faceVertexUvs[0] = []
  const edgeScale = 1

  for (let i = 0; i < faces.length; i++) {
    const v1 = rawGeometry.vertices[faces[i].a],
      v2 = rawGeometry.vertices[faces[i].b],
      v3 = rawGeometry.vertices[faces[i].c]

    if (v1.y === min.y && v2.y === min.y && v3.y === min.y) {
      rawGeometry.faceVertexUvs[0].push([
        new THREE.Vector2((v1.x + offset.x) / range.x, (v1.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v2.x + offset.x) / range.x, (v2.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v3.x + offset.x) / range.x, (v3.z + offset.z) / (range.z * edgeScale))
      ])
    } else if (v1.y === max.y && v2.y === max.y && v3.y === max.y) {
      rawGeometry.faceVertexUvs[0].push([
        new THREE.Vector2((v1.x + offset.x) / range.x, (v1.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v2.x + offset.x) / range.x, (v2.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v3.x + offset.x) / range.x, (v3.z + offset.z) / (range.z * edgeScale))
      ])
    } else if (v1.x === min.x && v2.x === min.x && v3.x === min.x) {
      rawGeometry.faceVertexUvs[0].push([
        new THREE.Vector2((v1.y + offset.y) / range.y, (v1.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v2.y + offset.y) / range.y, (v2.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v3.y + offset.y) / range.y, (v3.z + offset.z) / (range.z * edgeScale))
      ])
    } else if (v1.x === max.x && v2.x === max.x && v3.x === max.x) {
      rawGeometry.faceVertexUvs[0].push([
        new THREE.Vector2((v1.y + offset.y) / range.y, (v1.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v2.y + offset.y) / range.y, (v2.z + offset.z) / (range.z * edgeScale)),
        new THREE.Vector2((v3.y + offset.y) / range.y, (v3.z + offset.z) / (range.z * edgeScale))
      ])
    } else {
      rawGeometry.faceVertexUvs[0].push([
        new THREE.Vector2((v1.x + offset.x) / range.x, (v1.y + offset.y) / range.y),
        new THREE.Vector2((v2.x + offset.x) / range.x, (v2.y + offset.y) / range.y),
        new THREE.Vector2((v3.x + offset.x) / range.x, (v3.y + offset.y) / range.y)
      ])
    }
  }

  rawGeometry.uvsNeedUpdate = true
  const stlMesh = new THREE.Mesh(rawGeometry, material)
  return { stlMesh, min, max }
}


</script>


<template>
  <div>
    <canvas id="bg" style="width: 100vw; height: 100vh; position: fixed; top: 0; left: 0;"></canvas>
  </div>
</template>
