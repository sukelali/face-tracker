<template>
  <div class="container">
    <video ref="video" class="video-feed" autoplay muted playsinline></video>
    <div ref="threeCanvas" class="three-container"></div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';
import { FaceMesh } from '@mediapipe/face_mesh';
import { Camera } from '@mediapipe/camera_utils';

// Refs
const video = ref(null);
const threeCanvas = ref(null);

let targetPosition = new THREE.Vector3();
let currentPosition = new THREE.Vector3();
let targetRotation = new THREE.Euler();
let currentRotation = new THREE.Euler();

// Three.js variables
let scene, camera, renderer, sunglasses, faceMesh;

// MediaPipe landmarks
const LEFT_EYE_INDEX = 33;
const RIGHT_EYE_INDEX = 263;
const NOSE_BRIDGE_INDEX = 168;

onMounted(() => {
  initThreeJS();
  initFaceTracking();
});

function initThreeJS() {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 0.5; // Adjusted for better positioning

  renderer = new THREE.WebGLRenderer({ 
    alpha: true,
    antialias: true,
    preserveDrawingBuffer: true
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  threeCanvas.value.appendChild(renderer.domElement);

  // Lighting setup
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
  scene.add(ambientLight);
  
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
  directionalLight.position.set(0.5, 1, 0.5);
  scene.add(directionalLight);

  loadSunglassesModel();
}

function loadSunglassesModel() {
  const loader = new GLTFLoader();
  loader.load('/models/sunglass-1.glb', (gltf) => {
    sunglasses = gltf.scene;
    
    // Calculate initial scale based on model size
    const bbox = new THREE.Box3().setFromObject(sunglasses);
    const size = bbox.getSize(new THREE.Vector3());
    const scale = 0.8 / size.x; // Base scale factor
    // const scale = 21.80; // Base scale factor

    sunglasses.scale.set(scale, -scale, scale);
    
    // Center the model
    const center = bbox.getCenter(new THREE.Vector3());
    sunglasses.position.sub( center.multiplyScalar(scale) );
    
    // Initial rotation to face camera
    sunglasses.rotation.set(0, 0, 0);
    
    scene.add(sunglasses);
  }, undefined, (error) => {
    console.error('Error loading model:', error);
  });
}

function initFaceTracking() {
  faceMesh = new FaceMesh({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`
  });

  faceMesh.setOptions({
    maxNumFaces: 1,
    refineLandmarks: true,
    minDetectionConfidence: 0.7,
    minTrackingConfidence: 0.5
  });

  faceMesh.onResults((results) => {
    if (results.multiFaceLandmarks && sunglasses) {
      updateSunglassesPosition(results.multiFaceLandmarks[0]);
    }
  });

  new Camera(video.value, {
    onFrame: async () => await faceMesh.send({ image: video.value }),
    width: 640,
    height: 480
  }).start();
}

function updateSunglassesPosition(landmarks) {
  const leftEye = landmarks[LEFT_EYE_INDEX];
  const rightEye = landmarks[RIGHT_EYE_INDEX];
  const noseBridge = landmarks[NOSE_BRIDGE_INDEX];

  const mirroredLeftEyeX  = 1 - leftEye.x;
  const mirroredRightEyeX = 1 - rightEye.x;
 
  // Calculate eye center in normalized coordinates
  const eyeCenterX = (mirroredLeftEyeX + mirroredRightEyeX) / 2;
  const eyeCenterY = (leftEye.y + rightEye.y) / 2;

  // Convert to Three.js coordinates with proper scaling
  const x = (eyeCenterX - 0.5) *  0.62 - 0.31;
  const y = -(eyeCenterY - 0.5) * 0.22 + 0.11; // Adjusted downward
  const z = -noseBridge.z * 0.5 - 1.04; // Adjusted to match your z position

  // Calculate eye distance for dynamic scaling (if needed)
  const eyeDist = Math.hypot(mirroredRightEyeX - mirroredLeftEyeX, rightEye.y - leftEye.y);
  const scale = eyeDist * 43.6; // Adjusted to maintain ~21.80 scale
  
  // Calculate head tilt angle (with mirrored x coordinates)
  const angle = Math.atan2(rightEye.y - leftEye.y, mirroredRightEyeX - mirroredLeftEyeX);

  // Apply transformations
  sunglasses.position.set(x, y, z);
  sunglasses.scale.set(scale, scale, scale);
  sunglasses.rotation.set(
    Math.PI,           // x rotation (keep at 0 to match your perfect alignment)
    -angle * 0.5, // reduced y rotation
    -angle       // z rotation (inverted to match mirrored video)
  );

  console.log('Current rotation:', {
    x: sunglasses.rotation.x.toFixed(2),
    y: sunglasses.rotation.y.toFixed(2),
    z: sunglasses.rotation.z.toFixed(2)
  });

  renderer.render(scene, camera);
}
</script>

<style scoped>
.container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.video-feed {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0.5;
  transform: scaleX(-1); /* Mirror effect */
}

.three-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}
</style>
