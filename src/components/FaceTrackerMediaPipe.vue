<template>
  <div class="face-tracker">
    <video ref="video" class="input-video" autoplay muted playsinline></video>
    <canvas ref="canvas" class="output-canvas"></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { FaceMesh, FACEMESH_RIGHT_EYE, FACEMESH_LEFT_EYE, FACEMESH_LIPS, FACEMESH_TESSELATION } from '@mediapipe/face_mesh';
import { Camera } from '@mediapipe/camera_utils';
import { drawConnectors } from '@mediapipe/drawing_utils';

const video = ref(null);
const canvas = ref(null);

let faceMesh;

onMounted(() => {
  initFaceTracking();
});

function initFaceTracking() {
  // Initialize MediaPipe FaceMesh
  faceMesh = new FaceMesh({
    locateFile: (file) => {
      console.log('Requested file:', file);
      return `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`;
    }
  });

  faceMesh.setOptions({
    maxNumFaces: 1,
    refineLandmarks: true,
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5,
  });

  faceMesh.onResults((results) => {
    // Draw face landmarks on canvas
    const ctx = canvas.value.getContext('2d');
    ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);
    
    if (results.multiFaceLandmarks) {
      for (const landmarks of results.multiFaceLandmarks) {
        // Draw facial contours
        drawConnectors(ctx, landmarks, FACEMESH_TESSELATION, {
          color: '#C0C0F070',
          lineWidth: 1,
        });
        // Draw important landmarks (e.g., eyes, lips)
        drawConnectors(ctx, landmarks, FACEMESH_LIPS, { color: '#FF0000', lineWidth: 2 });
        drawConnectors(ctx, landmarks, FACEMESH_LEFT_EYE, { color: '#00FF00', lineWidth: 2 });
        drawConnectors(ctx, landmarks, FACEMESH_RIGHT_EYE, { color: '#00FF00', lineWidth: 2 });

      }
    }
  });

  // Start webcam
  const camera = new Camera(video.value, {
    onFrame: async () => {
      await faceMesh.send({ image: video.value });
    },
    width: 640,
    height: 480,
  });
  camera.start();
}

</script>

<style scoped>
.face-tracker {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.input-video {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  transform: scaleX(1); /* Mirror effect */
}

.output-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}
</style>