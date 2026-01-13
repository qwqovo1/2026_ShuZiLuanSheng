<template>
  <div ref="container" style="width: 100%; height: 100%"></div>
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount, defineEmits } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

const emit = defineEmits(['loading', 'progress'])

const container = ref(null)
let scene, camera, renderer, currentModel = null
let animationId = null
let isLoading = false
let modelCache = {}
let controls = null // 控制器实例

// 初始化 Three.js
onMounted(() => {
  initScene()
  loadModel('hunyuan01.glb').then(() => {
    preloadModel('hunyuan02.glb')
  })
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  if (animationId) cancelAnimationFrame(animationId)
  if (renderer) renderer.dispose()
  if (controls) controls.dispose()
})

function initScene() {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x1a1a1a)

  // 添加环境光（防止模型全黑）
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5)
  scene.add(ambientLight)

  // 添加方向光（模拟太阳光）
  const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
  directionalLight.position.set(1, 1, 1)
  scene.add(directionalLight)

  // 创建相机
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 0, 5)

  // 创建渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  container.value.appendChild(renderer.domElement)

  // 添加轨道控制器（鼠标交互）
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true // 平滑阻尼
  controls.dampingFactor = 0.05
  controls.rotateSpeed = 0.5
  controls.minDistance = 2
  controls.maxDistance = 10

  // 启动动画循环
  animate()
}

function handleResize() {
  if (!camera || !renderer) return
  camera.aspect = window.innerWidth / window.innerHeight
  camera.updateProjectionMatrix()
  renderer.setSize(window.innerWidth, window.innerHeight)
}

function animate() {
  animationId = requestAnimationFrame(animate)
  controls.update() // 更新控制器状态
  renderer.render(scene, camera)
}

// 预加载模型
function preloadModel(modelName) {
  if (modelCache[modelName] || isLoading) return

  const loader = new GLTFLoader()
  loader.load(
    `/model/${modelName}`,
    (gltf) => {
      modelCache[modelName] = gltf.scene.clone()
      console.log(`✅ 预加载完成: ${modelName}`)
    },
    undefined,
    (error) => {
      console.warn(`预加载失败: ${modelName}`, error)
    }
  )
}

// 加载模型（带进度、重试、缓存）
async function loadModel(modelName, retryCount = 0) {
  if (isLoading) return Promise.resolve()
  isLoading = true
  emit('loading', true)
  emit('progress', 0)

  if (modelCache[modelName]) {
    switchToModel(modelCache[modelName])
    isLoading = false
    emit('loading', false)
    return Promise.resolve()
  }

  const loader = new GLTFLoader()
  const oldModel = currentModel

  return new Promise((resolve, reject) => {
    const loadHandler = (gltf) => {
      const clonedScene = gltf.scene.clone()
      modelCache[modelName] = clonedScene
      switchToModel(clonedScene)
      isLoading = false
      emit('loading', false)
      resolve()
    }

    const progressHandler = (xhr) => {
      if (xhr.lengthComputable) {
        const percent = (xhr.loaded / xhr.total) * 100
        emit('progress', percent)
      }
    }

    const errorHandler = (error) => {
      if (retryCount < 2) {
        setTimeout(() => {
          loadModel(modelName, retryCount + 1).then(resolve).catch(reject)
        }, 1000)
      } else {
        console.error('模型最终加载失败:', error)
        alert(`❌ 模型 ${modelName} 加载失败，请刷新重试。`)
        if (oldModel && !scene.getObjectById(oldModel.id)) {
          scene.add(oldModel)
          currentModel = oldModel
        }
        isLoading = false
        emit('loading', false)
        emit('progress', 0)
        reject(error)
      }
    }

    loader.load(
      `/model/${modelName}`,
      loadHandler,
      progressHandler,
      errorHandler
    )
  })
}

function switchToModel(newModel) {
  if (currentModel) {
    scene.remove(currentModel)
  }
  currentModel = newModel
  scene.add(currentModel)
  // 重置控制器位置
  controls.reset()
}

// 双击切换模型（循环）
window.addEventListener('dblclick', () => {
  const nextModel = currentModel?.userData.modelName === 'hunyuan02.glb'
    ? 'hunyuan01.glb'
    : 'hunyuan02.glb'

  // 标记当前模型名
  if (currentModel) {
    currentModel.userData.modelName = nextModel
  }

  loadModel(nextModel)
})
</script>