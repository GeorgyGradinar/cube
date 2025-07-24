<template>
  <div
    ref="target"
    class="wrapper-3d-model"
  >
  </div>

  <button @click="reloadDice">reload</button>
</template>

<script setup>
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import * as CANNON from "cannon-es";
import CannonDebugger from "cannon-es-debugger";

const props = defineProps({
  dataModel: Object,
});

const target = ref(null);
let renderer;
let camera;
let previousePosition = ref(0);
let isDiceSettled = ref(false);
let updateDice = null;
let scene
let world
let lastDicePosition = ref(null);
const loadingPercent = ref(0);
let dice
let diceBody
let diceShape
const dropPatterns = [
  {
    position: { x: 0, y: 5, z: 0 },
    velocity: { x: 2, y: -10, z: 0 },
    angularVelocity: { x: 4, y: 0, z: 0 }
  },
  {
    position: { x: 0, y: 5, z: 0 },
    velocity: { x: -2, y: -10, z: 0 },
    angularVelocity: { x: -4, y: 0, z: 0 }
  },
  {
    position: { x: 0, y: 5, z: 0 },
    velocity: { x: 0, y: -10, z: 0 },
    angularVelocity: { x: 0, y: 4, z: 0 }
  },
  {
    position: { x: 0, y: 5, z: 0 },
    velocity: { x: 0, y: -10, z: 0 },
    angularVelocity: { x: 0, y: 0, z: 4 },
    quaternion: { x: 0.5, y: 0.5, z: 0.5, w: 0.5 }
  },
  {
    position: { x: 0, y: 5, z: 0 },
    velocity: { x: 0, y: -10, z: 0 },
    angularVelocity: { x: 0, y: 0, z: -4 },
    quaternion: { x: -0.5, y: -0.5, z: -0.5, w: 0.5 }
  }
];
let currentPatternIndex = 0;

onMounted(() => {
  createScene();
});


const onWindowResize = () => {
  if (!target.value) return;

  camera.aspect = target.value.clientWidth / target.value.clientHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(target.value.clientWidth, target.value.clientHeight);
};

function createScene() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x777777);

  // Physics world setup
  world = new CANNON.World({
    gravity: new CANNON.Vec3(0, -9.82, 0)
  });
  world.broadphase = new CANNON.SAPBroadphase(world);
  world.allowSleep = true;

  // Floor setup
  const floorGeometry = new THREE.PlaneGeometry(10, 10);
  const floorMaterial = new THREE.MeshStandardMaterial({
    color: 0xff0000,
    roughness: 0.8,
    metalness: 0.2
  });
  const floor = new THREE.Mesh(floorGeometry, floorMaterial);
  floor.rotation.x = -Math.PI / 2;
  floor.position.y = -1;
  floor.receiveShadow = true;
  scene.add(floor);

  // Floor physics body
  const floorBody = new CANNON.Body({
    type: CANNON.Body.STATIC,
    shape: new CANNON.Plane()
  });
  floorBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  floorBody.position.set(0, -1, 0);
  world.addBody(floorBody);

  addWalls(floorMaterial);

  // Debug renderer
  const cannonDebugger = new CannonDebugger(scene, world);

  addCamera();

  addLights();

  uploadModel();

  renderer = new THREE.WebGLRenderer({ antialias: false });
  renderer.setSize(target.value.clientWidth, target.value.clientHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));
  renderer.shadowMap.enabled = true;

  if (!target.value.contains(renderer.domElement)) {
    target.value.appendChild(renderer.domElement);
  }

  window.addEventListener("resize", onWindowResize);

  // Define the render loop
  const renderLoop = () => {
    requestAnimationFrame(renderLoop);

    // Update physics only if dice hasn't settled
    if (!isDiceSettled.value) {
      world.step(1 / 60);
    }

    // Update debug renderer
    cannonDebugger.update();

    // Update dice position if it exists
    if (updateDice && !isDiceSettled.value) {
      updateDice();
    }

    renderer.render(scene, camera);
  };

  // Start the render loop
  console.log('Starting render loop');
  renderLoop();
}

function reloadDice() {
  const pattern = dropPatterns[currentPatternIndex];
  currentPatternIndex = (currentPatternIndex + 1) % dropPatterns.length;
  
  // Reset to pattern position
  dice.position.set(pattern.position.x, pattern.position.y, pattern.position.z);
  dice.rotation.set(0, 0, 0);
  
  // Reset physics
  isDiceSettled.value = false;
  previousePosition.value = null;
  
  // Update existing physics body
  diceBody.position.set(pattern.position.x, pattern.position.y, pattern.position.z);
  diceBody.velocity.set(pattern.velocity.x, pattern.velocity.y, pattern.velocity.z);
  diceBody.angularVelocity.set(pattern.angularVelocity.x, pattern.angularVelocity.y, pattern.angularVelocity.z);
  
  // Apply quaternion if provided
  if (pattern.quaternion) {
    diceBody.quaternion.set(pattern.quaternion.x, pattern.quaternion.y, pattern.quaternion.z, pattern.quaternion.w);
  } else {
    diceBody.quaternion.set(0, 0, 0, 1);
  }
  
  diceBody.wakeUp();
  diceBody.previousPosition.copy(diceBody.position);
  diceBody.previousQuaternion.copy(diceBody.quaternion);
}

function addWalls(floorMaterial) {
  // Add left wall
  const leftWallGeometry = new THREE.PlaneGeometry(10, 10);
  const leftWall = new THREE.Mesh(leftWallGeometry, floorMaterial);
  leftWall.rotation.y = Math.PI / 2;
  leftWall.position.set(-5, 4, 0);
  leftWall.receiveShadow = true;
  scene.add(leftWall);

  const leftWallBody = new CANNON.Body({
    type: CANNON.Body.STATIC,
    shape: new CANNON.Plane()
  });
  leftWallBody.quaternion.setFromEuler(0, Math.PI / 2, 0);
  leftWallBody.position.set(-5, 4, 0);
  world.addBody(leftWallBody);

  // Add right wall
  const rightWallGeometry = new THREE.PlaneGeometry(10, 10);
  const rightWall = new THREE.Mesh(rightWallGeometry, floorMaterial);
  rightWall.rotation.y = -Math.PI / 2;
  rightWall.position.set(5, 4, 0);
  rightWall.receiveShadow = true;
  scene.add(rightWall);

  const rightWallBody = new CANNON.Body({
    type: CANNON.Body.STATIC,
    shape: new CANNON.Plane()
  });
  rightWallBody.quaternion.setFromEuler(0, -Math.PI / 2, 0);
  rightWallBody.position.set(5, 4, 0);
  world.addBody(rightWallBody);

  // Add back wall
  const backWallGeometry = new THREE.PlaneGeometry(10, 10);
  const backWall = new THREE.Mesh(backWallGeometry, floorMaterial);
  backWall.position.set(0, 4, -5);
  backWall.receiveShadow = true;
  scene.add(backWall);

  const backWallBody = new CANNON.Body({
    type: CANNON.Body.STATIC,
    shape: new CANNON.Plane()
  });
  backWallBody.position.set(0, 4, -5);
  world.addBody(backWallBody);
}

function uploadModel() {
  const loader = new GLTFLoader();
  loader.load(
    "models/dice.glb",
    (gltf) => {
      dice = gltf.scene;
      dice.scale.set(1, 1, 1);
      // Fixed starting position
      dice.position.set(0, 5, 0);
      dice.castShadow = true;
      dice.receiveShadow = true;
      scene.add(dice);

      // Fixed starting rotation
      dice.rotation.set(0, 0, 0);

      diceShape = new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5));
      diceBody = new CANNON.Body({
        mass: 5,
        shape: diceShape,
        position: new CANNON.Vec3(0, 5, 0),
        material: new CANNON.Material({
          friction: 0.3,
          restitution: 0.3
        }),
        linearDamping: 0.1,
        angularDamping: 0.1
      });

      // Fixed initial velocities
      diceBody.velocity.set(0, -10, 0);
      diceBody.angularVelocity.set(2, 2, 2);
      world.addBody(diceBody);

      // Update dice position in render loop
      updateDice = () => {
        dice.position.copy(diceBody.position);
        dice.quaternion.copy(diceBody.quaternion);
        console.log('update position ' + diceBody.velocity.y)
        console.log(isDiceSettled.value)
        // Check if dice has settled
        if (!isDiceSettled.value &&
          previousePosition.value?.y === diceBody.velocity.y &&
          previousePosition.value?.x === diceBody.velocity.x &&
          previousePosition.value?.z === diceBody.velocity.z) {
          isDiceSettled.value = true;
          // Store the final position when dice settles
          lastDicePosition.value = {
            position: diceBody.position.clone(),
            quaternion: diceBody.quaternion.clone()
          };
          console.log('Dice has settled at position:', lastDicePosition.value);
        }
        previousePosition.value = JSON.parse(JSON.stringify(diceBody.velocity));
      };

      // Force initial render
      renderer.render(scene, camera);
    },
    (xhr) => {
      loadingPercent.value = (xhr.loaded / xhr.total) * 100;
    },
    (error) => {
      console.error("An error happened", error);
    }
  );
}

function addCamera() {
  camera = new THREE.PerspectiveCamera(
    100,
    target.value.clientWidth / target.value.clientHeight,
    0.2,
    900
  );

  camera.position.set(0, 2, 5.5); // Adjust camera position for better view
  camera.lookAt(0, 0, 0);
}

function addLights() {
  const directionalLight = new THREE.DirectionalLight(0xffffff, 5);
  directionalLight.position.set(1, 1, 1).normalize();
  directionalLight.castShadow = true;
  scene.add(directionalLight);

  const directionalLight2 = new THREE.DirectionalLight(0xffffff, 5);
  directionalLight2.position.set(-1, -1, -1).normalize();
  directionalLight2.castShadow = true;
  scene.add(directionalLight2);
}

onBeforeUnmount(() => {
  renderer.dispose();
  window.removeEventListener("resize", onWindowResize);
});
</script>

<style scoped lang="scss">
.wrapper-3d-model {
  width: 100%;
  min-height: 60vh;
  position: relative;

  .wrapper-loading-text {
    position: absolute;
    width: 100%;
    height: 100%;
    background-color: #000000aa;
    display: flex;
    justify-content: center;
    align-items: center;

    color: var(--primary-white);
  }
}
</style>
