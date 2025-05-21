<template>
  <div>
    <div ref="canvasContainer" class="model-viewer"></div>
    <div class="debug-panel" v-if="model">
      <h3>Model Transformations</h3>
      <div>
        <strong>Position:</strong>
        X: {{ modelPosition.x }}, 
        Y: {{ modelPosition.y }}, 
        Z: {{ modelPosition.z }}
      </div>
      <div>
        <strong>Rotation:</strong>
        X: {{ modelRotation.x }}°, 
        Y: {{ modelRotation.y }}°, 
        Z: {{ modelRotation.z }}°
      </div>
      <div>
        <strong>Scale:</strong>
        X: {{ modelScale.x }}, 
        Y: {{ modelScale.y }}, 
        Z: {{ modelScale.z }}
      </div>
      <button @click="logModelData">Log to Console</button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

const canvasContainer = ref(null);

// Add these variables
const modelPosition = ref({ x: 0, y: 0, z: 0 });
const modelRotation = ref({ x: 0, y: 0, z: 0 });
const modelScale = ref({ x: 1, y: 1, z: 1 });


let scene, camera, renderer, model, controls;

onMounted(() => {
  initThreeJS();
  loadModel();
  animate();
});

onUnmounted(() => {
  if (renderer) {
    renderer.dispose(); // Clean up
  }
});


function initThreeJS() {
  // 1. Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xf0f0f0);

  // 2. Camera
  camera = new THREE.PerspectiveCamera(
    75,
    canvasContainer.value.clientWidth / canvasContainer.value.clientHeight,
    0.1,
    1000
  );
  camera.position.z = 5;

  // 3. Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  canvasContainer.value.appendChild(renderer.domElement);

  // 4. Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
  directionalLight.position.set(1, 1, 1);
  scene.add(directionalLight);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true; // Smooth movement
  controls.dampingFactor = 0.05;

  // Handle window resize
  window.addEventListener('resize', onWindowResize);
}

function loadModel() {
  const loader = new GLTFLoader();

  // Replace '/model.glb' with your model path
  loader.load(
    '/models/sunglass-1.glb',
    (gltf) => {
      model = gltf.scene;
      scene.add(model);
      
      const bbox = new THREE.Box3().setFromObject(model);
      const center = bbox.getCenter(new THREE.Vector3());
      const size = bbox.getSize(new THREE.Vector3());

      // Scale factor to fit the model in the camera's view
      const maxDim = Math.max(size.x, size.y, size.z);
      console.log('Model size:', maxDim);
      const scale = 4 / maxDim; 
      model.scale.set(scale, scale, scale);

      // Center the model
      model.position.sub(center).multiplyScalar(scale);

      console.log('Model loaded:', model);
    },
    undefined,
    (error) => {
      console.error('Error loading model:', error);
    }
  );
}

function animate() {
  requestAnimationFrame(animate);
  if (controls) controls.update(); // Required for damping

   if (model) {
    modelPosition.value = {
      x: model.position.x.toFixed(2),
      y: model.position.y.toFixed(2),
      z: model.position.z.toFixed(2)
    };
    
    modelRotation.value = {
      x: THREE.MathUtils.radToDeg(model.rotation.x).toFixed(2),
      y: THREE.MathUtils.radToDeg(model.rotation.y).toFixed(2),
      z: THREE.MathUtils.radToDeg(model.rotation.z).toFixed(2)
    };
    
    modelScale.value = {
      x: model.scale.x.toFixed(2),
      y: model.scale.y.toFixed(2),
      z: model.scale.z.toFixed(2)
    };

    console.log('Model Position:', modelPosition.value);
    console.log('Model Rotation:', modelRotation.value);
    console.log('Model Scale:', modelScale.value);
  }

  renderer.render(scene, camera);
}


function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

function logModelData() {
    if (this.model) {
      console.log('Current Model Transform:');
      console.log('Position:', this.model.position);
      console.log('Rotation:', {
        x: THREE.MathUtils.radToDeg(this.model.rotation.x),
        y: THREE.MathUtils.radToDeg(this.model.rotation.y),
        z: THREE.MathUtils.radToDeg(this.model.rotation.z)
      });
      console.log('Scale:', this.model.scale);
    }
  }


</script>

<style scoped>
.model-viewer {
  width: 100vw;  /* Full viewport width */
  height: 100vh; /* Full viewport height */
  position: fixed;
  top: 0;
  left: 0;
}
</style>
