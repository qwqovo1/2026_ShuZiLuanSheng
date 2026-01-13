<template>
  <div ref="container" class="model-viewer"></div>
</template>

<script setup>
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { onMounted, onUnmounted, ref } from 'vue';

const container = ref(null);
let scene, camera, renderer, controls, currentModel = null;
let isShowingFirst = true;

const init = () => {
  // 场景
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xf5f7fa);

  // 相机
  camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 2, 5);

  // 渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  container.value.appendChild(renderer.domElement);

  // 控制器（允许鼠标旋转/缩放）
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;

  // 灯光
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(5, 10, 7);
  dirLight.castShadow = true;
  scene.add(dirLight);

  // 初始加载第一个模型
  loadModel('/model/hunyuan01.glb');

  // 事件监听
  window.addEventListener('resize', onWindowResize);
  window.addEventListener('click', toggleModel);
};

const loadModel = (path) => {
  // 清除旧模型
  if (currentModel) {
    scene.remove(currentModel);
    currentModel = null;
  }

  const loader = new GLTFLoader();
  loader.load(
    path,
    (gltf) => {
      currentModel = gltf.scene;
      // 自动缩放居中
      const box = new THREE.Box3().setFromObject(currentModel);
      const center = box.getCenter(new THREE.Vector3());
      const size = box.getSize(new THREE.Vector3()).length();
      const scale = 3 / size;
      currentModel.scale.set(scale, scale, scale);
      currentModel.position.sub(center.clone().multiplyScalar(scale));
      scene.add(currentModel);
    },
    undefined,
    (error) => {
      console.error('❌ 模型加载失败:', error);
    }
  );
};

const toggleModel = () => {
  isShowingFirst = !isShowingFirst;
  const path = isShowingFirst ? '/model/hunyuan01.glb' : '/model/hunyuan02.glb';
  loadModel(path);
};

const onWindowResize = () => {
  if (!camera || !renderer) return;
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
};

const animate = () => {
  requestAnimationFrame(animate);
  if (controls) controls.update();
  renderer.render(scene, camera);
};

const dispose = () => {
  window.removeEventListener('resize', onWindowResize);
  window.removeEventListener('click', toggleModel);
  if (renderer) {
    renderer.dispose();
    if (container.value && renderer.domElement) {
      container.value.removeChild(renderer.domElement);
    }
  }
};

onMounted(() => {
  init();
  animate();
});

onUnmounted(() => {
  dispose();
});
</script>

<style scoped>
.model-viewer {
  width: 100%;
  height: 100%;
  display: block;
}
</style>