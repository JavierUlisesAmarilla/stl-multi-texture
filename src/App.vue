<script setup lang="ts">

import * as THREE from "three";
import { STLLoader } from 'three/examples/jsm/loaders/STLLoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { onMounted } from "vue";

const scene = new THREE.Scene();
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let geometry: THREE.BufferGeometry;
let material: THREE.MeshPhongMaterial;
let mesh: THREE.Mesh;

onMounted(() => {
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

  loadModel();

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

function loadModel() {
  const loader = new STLLoader()
  loader.load(
    "/model.stl",
    (data: any) => {
      geometry = data;
      addModel();
    },
    (xhr: any) => {
      console.log(Math.ceil((xhr.loaded / xhr.total) * 100))
    },
    (error: any) => {
      console.log(error)
    }
  );
}

function addModel() {
  if (geometry) {
    const selectedObject = scene.getObjectByName("object");
    if (selectedObject)
      scene.remove(selectedObject);

    geometry.center()
    geometry.computeVertexNormals();

    material = new THREE.MeshPhongMaterial({ color: 0xBE975B });
    mesh = new THREE.Mesh(geometry, material)
    mesh.rotation.x = -Math.PI / 2;

    mesh.name = "object";
    scene.add(mesh)

    fitCameraToSelection(camera, controls, mesh, 1.2);
  }
}

</script>

<template>
  <div>
    <canvas id="bg" class="" width="800" height="800"></canvas>
  </div>
</template>
