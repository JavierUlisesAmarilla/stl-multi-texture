<script setup lang="ts">

import * as THREE from "three";
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'
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

  getModelMesh('/models/model.stl', '/textures/cherry.jpg').then((modelMesh) => {
    modelMesh.rotation.x = -Math.PI / 2;
    scene.add(modelMesh)
    fitCameraToSelection(camera, controls, modelMesh, 1.2);
  })

  animate();
});

function animate() {
  controls.update()
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}

const size = new THREE.Vector3();
const center = new THREE.Vector3();
const box = new THREE.Box3();

function fitCameraToSelection(camera: any, controls: any, selection: any, fitOffset = 1.2) {
  box.makeEmpty();
  box.expandByObject(selection);

  box.getSize(size);
  box.getCenter(center);

  const maxSize = Math.max(size.x, size.y, size.z);
  const fitHeightDistance = maxSize / (2 * Math.atan(Math.PI * camera.fov / 360));
  const fitWidthDistance = fitHeightDistance / camera.aspect;
  const distance = fitOffset * Math.max(fitHeightDistance, fitWidthDistance);

  const direction = controls.target.clone()
    .sub(camera.position)
    .normalize()
    .multiplyScalar(distance);

  controls.maxDistance = distance * 10;
  controls.target.copy(center);

  camera.near = distance / 100;
  camera.far = distance * 100;
  camera.updateProjectionMatrix();

  camera.position.copy(controls.target).sub(direction);

  controls.update();
}

async function getModelMesh(modelPath: string, texturePath: string) {
  const stlLoader = new STLLoader()
  const textureLoader = new THREE.TextureLoader()
  const stlGeo = await stlLoader.loadAsync(modelPath)
  const stlMaterial = new THREE.MeshBasicMaterial({
    map: textureLoader.load(texturePath),
    side: THREE.DoubleSide,
  })
  const modelMesh = getStlMesh(stlGeo, stlMaterial)
  return modelMesh
}

function getStlMesh(geometry: any, material: any) {
  let rawGeometry = geometry
  if (geometry.isBufferGeometry) {
    rawGeometry = new THREE.Geometry().fromBufferGeometry(geometry)
  }
  rawGeometry.computeBoundingBox();
  const max = rawGeometry.boundingBox.max,
    min = rawGeometry.boundingBox.min;
  const offset = new THREE.Vector2(0 - min.x, 0 - min.y);
  const range = new THREE.Vector2(max.x - min.x, max.y - min.y);
  const faces = rawGeometry.faces;
  rawGeometry.faceVertexUvs[0] = [];

  for (let i = 0; i < faces.length; i++) {
    const v1 = rawGeometry.vertices[faces[i].a],
      v2 = rawGeometry.vertices[faces[i].b],
      v3 = rawGeometry.vertices[faces[i].c];
    rawGeometry.faceVertexUvs[0].push([
      new THREE.Vector2((v1.x + offset.x) / range.x, (v1.y + offset.y) / range.y),
      new THREE.Vector2((v2.x + offset.x) / range.x, (v2.y + offset.y) / range.y),
      new THREE.Vector2((v3.x + offset.x) / range.x, (v3.y + offset.y) / range.y)
    ]);
  }

  rawGeometry.uvsNeedUpdate = true;

  const stlMesh = new THREE.Mesh(rawGeometry, material)
  return stlMesh
}

</script>

<template>
  <div>
    <canvas id="bg" class="" width="800" height="800"></canvas>
  </div>
</template>
