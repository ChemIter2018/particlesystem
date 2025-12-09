<template>
  <div class="scene-container">
    <video ref="videoElement" class="input_video"></video>
    <canvas ref="debugCanvas" class="debug-canvas"></canvas>
    <div ref="canvasContainer" class="canvas-container"></div>
    
    <!-- Loading Overlay -->
    <div v-if="loading" class="loading-overlay">
      <div class="loader-container">
        <div class="loader-spinner"></div>
        <div class="loader-text">正在加载模型...</div>
      </div>
    </div>
    
    <!-- Gesture Hints -->
    <div class="gesture-hints" v-if="!loading">
      <div class="hint-title">手势控制</div>
      <div class="hint-item">
        <span class="icon">👐</span>
        <span class="text">张开 / 合拢：缩放</span>
      </div>
      <div class="hint-item">
        <span class="icon">👈👉</span>
        <span class="text">左右移动：旋转</span>
      </div>
      <div class="hint-item">
        <span class="icon">👆👇</span>
        <span class="text">上下移动：翻转</span>
      </div>
      <div class="hint-status" :class="{ active: handState.present }">
        {{ handState.present ? '✓ 检测到手势' : '○ 未检测到手' }}
      </div>
    </div>
    
    <!-- Info panel -->
    <div class="info-panel" v-if="!loading">
      <div class="info-item">缩放: {{ (handState.distance * 100).toFixed(0) }}%</div>
      <div class="info-item">旋转: {{ handState.rotationY.toFixed(0) }}°</div>
      <div class="info-item">倾斜: {{ handState.rotationX.toFixed(0) }}°</div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue';
import * as THREE from 'three';
import { Hands } from '@mediapipe/hands';
import { Camera } from '@mediapipe/camera_utils';
import { drawConnectors, drawLandmarks } from '@mediapipe/drawing_utils';
import { HAND_CONNECTIONS } from '@mediapipe/hands';
import GUI from 'lil-gui';

const videoElement = ref(null);
const debugCanvas = ref(null);
const canvasContainer = ref(null);
const loading = ref(true);

let scene, camera, renderer, particles;
let animationId;
let gui;
let cameraUtils;
let hands;
let canvasCtx;

// Configuration
const config = {
  particleCount: 3000,
  particleSize: 0.15,
  color: '#00ffff',
  shape: 'saturn',
  handinteraction: true,
  showCamera: true,
  rotationSensitivity: 2.0,
  tiltSensitivity: 1.5,
  scaleSensitivity: 2.0
};

// Enhanced hand tracking state
const handState = {
  distance: 1.0,        // Scale (0.3 - 2.5)
  openness: 0.5,        // Hand openness (0 = closed, 1 = open)
  centerX: 0.5,         // Hand center X position (0-1)
  centerY: 0.5,         // Hand center Y position (0-1)
  rotationY: 0,         // Y-axis rotation in degrees
  rotationX: 0,         // X-axis rotation in degrees
  present: false,
  prevCenterX: 0.5,
  prevCenterY: 0.5
};

const shapes = {
  heart: (i, count) => {
    const t = (i / count) * Math.PI * 2 * 10;
    const x = 16 * Math.pow(Math.sin(t), 3);
    const y = 13 * Math.cos(t) - 5 * Math.cos(2 * t) - 2 * Math.cos(3 * t) - Math.cos(4 * t);
    const z = (Math.random() - 0.5) * 5;
    return new THREE.Vector3(x * 0.2, y * 0.2, z);
  },
  flower: (i, count) => {
    const theta = (i / count) * Math.PI * 2 * 20;
    const r = 5 * Math.cos(5 * theta);
    const x = r * Math.cos(theta);
    const y = r * Math.sin(theta);
    const z = (Math.random() - 0.5) * 5;
    return new THREE.Vector3(x, y, z);
  },
  saturn: (i, count) => {
    if (i < count * 0.3) {
      const phi = Math.acos(-1 + (2 * i) / (count * 0.3));
      const theta = Math.sqrt((count * 0.3) * Math.PI) * phi;
      const r = 4;
      return new THREE.Vector3(
        r * Math.cos(theta) * Math.sin(phi),
        r * Math.sin(theta) * Math.sin(phi),
        r * Math.cos(phi)
      );
    } else {
      const angle = (i * 0.1);
      const radius = 6 + Math.random() * 4;
      return new THREE.Vector3(
        radius * Math.cos(angle),
        (Math.random() - 0.5) * 0.5,
        radius * Math.sin(angle)
      );
    }
  },
  fireworks: (i, count) => {
    const theta = Math.random() * Math.PI * 2;
    const phi = Math.acos((Math.random() * 2) - 1);
    const r = Math.random() * 10;
    return new THREE.Vector3(
      r * Math.sin(phi) * Math.cos(theta),
      r * Math.sin(phi) * Math.sin(theta),
      r * Math.cos(phi)
    );
  },
  buddha: (i, count) => {
     // Simple sitting figure approximation
     // Body parts ratios
     const headCount = Math.floor(count * 0.15);
     const bodyCount = Math.floor(count * 0.45);
     const legsCount = count - headCount - bodyCount;

     if (i < headCount) {
        // Head (Sphere at top)
        const idx = i;
        const phi = Math.acos(-1 + (2 * idx) / headCount);
        const theta = Math.sqrt(headCount * Math.PI) * phi;
        const r = 1.2;
        return new THREE.Vector3(
            r * Math.cos(theta) * Math.sin(phi),
            4.5 + r * Math.sin(theta) * Math.sin(phi), // y offset
            r * Math.cos(phi)
        );
     } else if (i < headCount + bodyCount) {
        // Torso (Ellipsoid)
        const idx = i - headCount;
        const phi = Math.acos(-1 + (2 * idx) / bodyCount);
        const theta = Math.sqrt(bodyCount * Math.PI) * phi;
        return new THREE.Vector3(
            1.8 * Math.cos(theta) * Math.sin(phi),
            2 + 2.5 * Math.sin(theta) * Math.sin(phi),
            1.5 * Math.cos(phi)
        );
     } else {
        // Legs (Crossed - simplified as a wide base torus/disk sector)
        const idx = i - (headCount + bodyCount);
        const angle = (idx / legsCount) * Math.PI * 2;
        const r = 3 + Math.random();
        // flatten y
        return new THREE.Vector3(
            r * Math.cos(angle),
            0.5 * Math.sin(angle * 2) - 0.5, // curve up slightly at knees?
            r * Math.sin(angle)
        );
     }
  }
};

onMounted(() => {
  initThree();
  initMediaPipe();
  initGUI();
  animate();
  window.addEventListener('resize', onWindowResize);
});

onUnmounted(() => {
  if (cameraUtils) cameraUtils.stop();
  if (renderer) renderer.dispose();
  if (gui) gui.destroy();
  window.removeEventListener('resize', onWindowResize);
});

function initThree() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x050505);

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 20;

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  canvasContainer.value.appendChild(renderer.domElement);

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const pointLight = new THREE.PointLight(0xffffff, 1);
  pointLight.position.set(10, 10, 10);
  scene.add(pointLight);

  createParticles();
}

function createParticles() {
  if (particles) scene.remove(particles);

  const geometry = new THREE.BufferGeometry();
  const positions = [];
  const generator = shapes[config.shape] || shapes.saturn;

  for (let i = 0; i < config.particleCount; i++) {
    const v = generator(i, config.particleCount);
    positions.push(v.x, v.y, v.z);
  }

  geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));

  const material = new THREE.PointsMaterial({
    color: config.color,
    size: config.particleSize,
    sizeAttenuation: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    transparent: true,
    opacity: 0.8
  });

  particles = new THREE.Points(geometry, material);
  scene.add(particles);
}

function initMediaPipe() {
  hands = new Hands({
    locateFile: (file) => {
      return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
    }
  });

  hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 1,
    minDetectionConfidence: 0.7,
    minTrackingConfidence: 0.7
  });

  hands.onResults(onResults);

  if (videoElement.value && debugCanvas.value) {
    canvasCtx = debugCanvas.value.getContext('2d');
    
    const cameraMedia = new Camera(videoElement.value, {
      onFrame: async () => {
        if (videoElement.value) {
           await hands.send({ image: videoElement.value });
        }
      },
      width: 640,
      height: 480
    });
  
    cameraUtils = cameraMedia;
    cameraMedia.start().then(() => {
        loading.value = false;
    }).catch(err => {
        console.error("Camera error:", err);
        loading.value = false;
    });
  }
}

function calculateHandOpenness(landmarks) {
  // Calculate hand openness based on finger spread
  // Compare distances between fingertips and palm center
  const wrist = landmarks[0];
  const palmCenter = landmarks[9]; // Middle finger base
  
  // Get fingertips: thumb, index, middle, ring, pinky
  const tips = [4, 8, 12, 16, 20].map(i => landmarks[i]);
  
  // Calculate average distance from tips to palm center
  let totalDist = 0;
  tips.forEach(tip => {
    const dx = tip.x - palmCenter.x;
    const dy = tip.y - palmCenter.y;
    const dz = tip.z - palmCenter.z;
    totalDist += Math.sqrt(dx*dx + dy*dy + dz*dz);
  });
  
  const avgDist = totalDist / 5;
  
  // Normalize to 0-1 range (empirically determined thresholds)
  // Closed fist: ~0.05-0.1, Open hand: ~0.2-0.3
  const openness = Math.max(0, Math.min(1, (avgDist - 0.05) / 0.25));
  
  return openness;
}

function getHandCenter(landmarks) {
  // Calculate center of hand (average of wrist and middle finger tip)
  const wrist = landmarks[0];
  const middleTip = landmarks[12];
  
  return {
    x: (wrist.x + middleTip.x) / 2,
    y: (wrist.y + middleTip.y) / 2,
    z: (wrist.z + middleTip.z) / 2
  };
}

function onResults(results) {
  // Clear and draw debug canvas
  if (canvasCtx && debugCanvas.value) {
    canvasCtx.save();
    canvasCtx.clearRect(0, 0, debugCanvas.value.width, debugCanvas.value.height);
    canvasCtx.drawImage(results.image, 0, 0, debugCanvas.value.width, debugCanvas.value.height);
    
    if (results.multiHandLandmarks) {
      for (const landmarks of results.multiHandLandmarks) {
        drawConnectors(canvasCtx, landmarks, HAND_CONNECTIONS, {color: '#00FF00', lineWidth: 2});
        drawLandmarks(canvasCtx, landmarks, {color: '#FF0000', lineWidth: 1, radius: 3});
      }
    }
    canvasCtx.restore();
  }
  
  if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
    handState.present = true;
    
    const landmarks = results.multiHandLandmarks[0];
    
    // 1. Calculate hand openness (for scaling)
    const openness = calculateHandOpenness(landmarks);
    handState.openness = openness;
    
    // Map openness to scale: closed (0) -> 0.3x, open (1) -> 2.5x
    const targetScale = 0.3 + openness * 2.2 * config.scaleSensitivity;
    handState.distance += (targetScale - handState.distance) * 0.15;
    
    // 2. Get hand center position
    const center = getHandCenter(landmarks);
    handState.centerX = center.x;
    handState.centerY = center.y;
    
    // 3. Calculate rotation based on horizontal movement
    // Map x position (0-1) to rotation (-180 to 180 degrees)
    const targetRotationY = ((center.x - 0.5) * 360 * config.rotationSensitivity);
    handState.rotationY += (targetRotationY - handState.rotationY) * 0.1;
    
    // 4. Calculate tilt based on vertical movement
    // Map y position (0-1) to tilt (-90 to 90 degrees)
    // Note: y=0 is top, y=1 is bottom, so we invert
    const targetRotationX = ((0.5 - center.y) * 180 * config.tiltSensitivity);
    handState.rotationX += (targetRotationX - handState.rotationX) * 0.1;
    
    // Update previous positions
    handState.prevCenterX = center.x;
    handState.prevCenterY = center.y;
    
  } else {
    handState.present = false;
    // Smoothly return to default state
    handState.distance += (1.0 - handState.distance) * 0.05;
    handState.rotationY *= 0.95;
    handState.rotationX *= 0.95;
    handState.openness += (0.5 - handState.openness) * 0.05;
  }
}

function initGUI() {
  gui = new GUI({ title: '控制面板' });
  
  const shapeFolder = gui.addFolder('粒子形状');
  shapeFolder.add(config, 'shape', { 
    '心形': 'heart', 
    '花朵': 'flower', 
    '土星': 'saturn', 
    '佛像': 'buddha', 
    '烟花': 'fireworks' 
  }).name('模板').onChange(createParticles);
  shapeFolder.addColor(config, 'color').name('颜色').onChange((v) => {
    if(particles && !handState.present) particles.material.color.set(v);
  });
  shapeFolder.add(config, 'particleCount', 500, 10000, 100).name('数量').onFinishChange(createParticles);
  shapeFolder.add(config, 'particleSize', 0.05, 0.5).name('大小').onChange((v) => {
    if(particles) particles.material.size = v;
  });
  shapeFolder.open();
  
  const gestureFolder = gui.addFolder('手势控制');
  gestureFolder.add(config, 'handinteraction').name('启用手势');
  gestureFolder.add(config, 'scaleSensitivity', 0.5, 3.0).name('缩放灵敏度');
  gestureFolder.add(config, 'rotationSensitivity', 0.5, 4.0).name('旋转灵敏度');
  gestureFolder.add(config, 'tiltSensitivity', 0.5, 3.0).name('翻转灵敏度');
  gestureFolder.open();
  
  const displayFolder = gui.addFolder('显示设置');
  displayFolder.add(config, 'showCamera').name('显示摄像头').onChange((v) => {
    if (videoElement.value) {
      videoElement.value.style.display = v ? 'block' : 'none';
    }
    if (debugCanvas.value) {
      debugCanvas.value.style.display = v ? 'block' : 'none';
    }
  });
  displayFolder.open();
}

function animate() {
  requestAnimationFrame(animate);
  
  if (particles) {
     if (config.handinteraction && handState.present) {
        // Apply scaling based on hand openness
        const targetScale = handState.distance;
        particles.scale.lerp(new THREE.Vector3(targetScale, targetScale, targetScale), 0.15);
        
        // Apply rotation based on hand position
        // Y-axis rotation (left-right movement)
        const targetRotY = (handState.rotationY * Math.PI) / 180;
        particles.rotation.y += (targetRotY - particles.rotation.y) * 0.1;
        
        // X-axis rotation (up-down movement for tilt/flip)
        const targetRotX = (handState.rotationX * Math.PI) / 180;
        particles.rotation.x += (targetRotX - particles.rotation.x) * 0.1;
        
        // Subtle continuous rotation for visual appeal
        particles.rotation.y += 0.001;
        
        // Color effect based on openness
        const hue = 180 + handState.openness * 180; // Cyan to Magenta
        particles.material.color.setHSL(hue / 360, 1.0, 0.5);
        
     } else {
        // Auto-rotate when no hand detected
        particles.rotation.y += 0.003;
        particles.rotation.x += 0.001;
        
        // Gentle pulsing
        const time = Date.now() * 0.001;
        const pulse = 1 + Math.sin(time * 0.5) * 0.1;
        particles.scale.lerp(new THREE.Vector3(pulse, pulse, pulse), 0.05);
        
        // Restore default color
        particles.material.color.set(config.color);
     }
  }

  renderer.render(scene, camera);
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;600;700&display=swap');

.scene-container {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: radial-gradient(ellipse at bottom, #0d1117 0%, #010409 100%);
  font-family: 'Inter', sans-serif;
}

.canvas-container {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  z-index: 1;
}

/* Camera preview with glassmorphism effect */
.input_video {
  position: absolute;
  bottom: 20px;
  right: 20px;
  width: 240px;
  height: 180px;
  border-radius: 16px;
  z-index: 10;
  border: 2px solid rgba(255, 255, 255, 0.1);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  transform: scaleX(-1);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.input_video:hover {
  border-color: rgba(0, 255, 255, 0.5);
  box-shadow: 0 12px 48px rgba(0, 255, 255, 0.3);
  transform: scaleX(-1) scale(1.05);
}

.debug-canvas {
  position: absolute;
  bottom: 20px;
  right: 20px;
  width: 240px;
  height: 180px;
  border-radius: 16px;
  z-index: 11;
  border: 2px solid rgba(0, 255, 255, 0.3);
  box-shadow: 0 8px 32px rgba(0, 255, 255, 0.2);
  transform: scaleX(-1);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.debug-canvas:hover {
  border-color: rgba(0, 255, 255, 0.6);
  box-shadow: 0 12px 48px rgba(0, 255, 255, 0.4);
  transform: scaleX(-1) scale(1.05);
}

/* Loading overlay with modern animation */
.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, #0d1117 0%, #010409 100%);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
  backdrop-filter: blur(10px);
}

.loader-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
}

.loader-spinner {
  width: 60px;
  height: 60px;
  border: 4px solid rgba(0, 255, 255, 0.1);
  border-top: 4px solid #00ffff;
  border-radius: 50%;
  animation: spin 1s cubic-bezier(0.68, -0.55, 0.265, 1.55) infinite;
  box-shadow: 0 0 20px rgba(0, 255, 255, 0.3);
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loader-text {
  color: #00ffff;
  font-size: 18px;
  font-weight: 600;
  letter-spacing: 0.5px;
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 0.6; }
  50% { opacity: 1; }
}

/* Gesture hints panel */
.gesture-hints {
  position: absolute;
  top: 20px;
  left: 20px;
  background: rgba(13, 17, 23, 0.7);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 20px;
  padding: 20px;
  z-index: 10;
  min-width: 260px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.gesture-hints:hover {
  background: rgba(13, 17, 23, 0.85);
  border-color: rgba(0, 255, 255, 0.3);
  box-shadow: 0 12px 48px rgba(0, 255, 255, 0.2);
  transform: translateY(-2px);
}

.hint-title {
  font-size: 20px;
  font-weight: 700;
  color: #00ffff;
  margin-bottom: 16px;
  text-align: center;
  letter-spacing: 1px;
  text-transform: uppercase;
  background: linear-gradient(135deg, #00ffff 0%, #00ccff 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.hint-item {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 12px;
  padding: 10px;
  background: rgba(255, 255, 255, 0.03);
  border-radius: 12px;
  transition: all 0.2s ease;
}

.hint-item:hover {
  background: rgba(0, 255, 255, 0.1);
  transform: translateX(4px);
}

.hint-item .icon {
  font-size: 24px;
  filter: drop-shadow(0 0 8px rgba(0, 255, 255, 0.5));
}

.hint-item .text {
  color: rgba(255, 255, 255, 0.9);
  font-size: 14px;
  font-weight: 400;
  letter-spacing: 0.3px;
}

.hint-status {
  margin-top: 12px;
  padding: 12px;
  text-align: center;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.05);
  color: rgba(255, 255, 255, 0.5);
  font-size: 13px;
  font-weight: 600;
  border: 1px solid rgba(255, 255, 255, 0.1);
  transition: all 0.3s ease;
}

.hint-status.active {
  background: rgba(0, 255, 100, 0.15);
  color: #00ff88;
  border-color: rgba(0, 255, 100, 0.3);
  box-shadow: 0 0 20px rgba(0, 255, 100, 0.2);
  animation: statusPulse 2s ease-in-out infinite;
}

@keyframes statusPulse {
  0%, 100% { opacity: 0.8; }
  50% { opacity: 1; }
}

/* Info panel */
.info-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  background: rgba(13, 17, 23, 0.7);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 20px;
  padding: 16px 20px;
  z-index: 10;
  min-width: 200px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.info-panel:hover {
  background: rgba(13, 17, 23, 0.85);
  border-color: rgba(0, 255, 255, 0.3);
  box-shadow: 0 12px 48px rgba(0, 255, 255, 0.2);
  transform: translateY(-2px);
}

.info-item {
  color: rgba(255, 255, 255, 0.9);
  font-size: 14px;
  font-weight: 500;
  margin-bottom: 8px;
  padding: 8px 12px;
  background: rgba(255, 255, 255, 0.03);
  border-radius: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  letter-spacing: 0.3px;
  transition: all 0.2s ease;
}

.info-item:hover {
  background: rgba(0, 255, 255, 0.1);
  transform: translateX(-4px);
}

.info-item:last-child {
  margin-bottom: 0;
}

/* Responsive design */
@media (max-width: 768px) {
  .gesture-hints {
    top: 10px;
    left: 10px;
    padding: 16px;
    min-width: 220px;
  }
  
  .info-panel {
    top: 10px;
    right: 10px;
    padding: 12px 16px;
    min-width: 160px;
  }
  
  .input_video, .debug-canvas {
    width: 160px;
    height: 120px;
    bottom: 10px;
    right: 10px;
  }
  
  .hint-title {
    font-size: 16px;
  }
  
  .hint-item .text {
    font-size: 12px;
  }
  
  .info-item {
    font-size: 12px;
  }
}

/* Smooth entrance animations */
.gesture-hints {
  animation: slideInLeft 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}

.info-panel {
  animation: slideInRight 0.6s cubic-bezier(0.4, 0, 0.2, 1);
}

.input_video, .debug-canvas {
  animation: fadeInScale 0.8s cubic-bezier(0.4, 0, 0.2, 1);
}

@keyframes slideInLeft {
  from {
    opacity: 0;
    transform: translateX(-30px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes slideInRight {
  from {
    opacity: 0;
    transform: translateX(30px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

@keyframes fadeInScale {
  from {
    opacity: 0;
    transform: scaleX(-1) scale(0.8);
  }
  to {
    opacity: 1;
    transform: scaleX(-1) scale(1);
  }
}
</style>
