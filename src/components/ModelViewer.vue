<!-- src/components/ModelViewer.vue -->
<template>
  <div ref="container" style="width: 100%; height: 100%"></div>
</template>

<script setup>
import { onMounted, ref, onBeforeUnmount, defineEmits } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'

const emit = defineEmits(['loading', 'progress'])

const container = ref(null)
let scene, camera, renderer, currentModel = null
let animationId = null
let isLoading = false
let modelCache = {} // 缓存已成功加载的模型

// 初始化 Three.js
onMounted(() => {
  initScene()
  loadModel('hunyuan01.glb').then(() => {
    // 首次加载成功后，预加载另一个模型（后台静默加载）
    preloadModel('hunyuan02.glb')
  })
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)
  if (animationId) cancelAnimationFrame(animationId)
  if (renderer) renderer.dispose()
})

function initScene() {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x1a1a1a)

  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 0, 5)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  container.value.appendChild(renderer.domElement)

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
  if (currentModel) {
    currentModel.rotation.y += 0.01
  }
  renderer.render(scene, camera)
}

// 预加载模型（不触发 loading 状态，不替换场景）
function preloadModel(modelName) {
  if (modelCache[modelName] || isLoading) return

  const loader = new GLTFLoader()
  loader.load(
    `/model/${modelName}`,
    (gltf) => {
      modelCache[modelName] = gltf.scene.clone()
      console.log(`✅ 预加载完成: ${modelName}`)
    },
    (xhr) => {
      // 可选：记录预加载进度（不暴露给 UI）
    },
    (error) => {
      console.warn(`预加载失败: ${modelName}`, error)
    }
  )
}

// 带重试、进度、缓存的主加载函数
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
        console.warn(`加载失败，第 ${retryCount + 1} 次重试...`, error)
        setTimeout(() => {
          loadModel(modelName, retryCount + 1).then(resolve).catch(reject)
        }, 1000)
      } else {
        console.error('模型最终加载失败:', error)
        alert(`❌ 模型 ${modelName} 加载失败，请刷新重试。`)
        // 恢复旧模型
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
}

// 点击切换
window.addEventListener('click', () => {
  const nextModel = currentModel?.userData.modelName === 'hunyuan02.glb'
    ? 'hunyuan01.glb'
    : 'hunyuan02.glb'

  // 标记当前模型名（用于判断）
  if (currentModel) {
    currentModel.userData.modelName = nextModel
  }

  loadModel(nextModel)
})
</script>