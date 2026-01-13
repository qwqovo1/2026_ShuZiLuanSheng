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
const modelCache = {}
let controls = null

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
  scene.background = new THREE.Color(0x222222) // 稍亮一点的背景，避免纯黑压抑

  // 🌤️ 温和柔光方案：HemisphereLight（天光+地光） + 微弱环境光
  const hemisphereLight = new THREE.HemisphereLight(
    0xffffff, // 天空颜色（偏白）
    0x444444, // 地面颜色（灰，模拟地面反光）
    0.8       // 整体强度（比默认0.5更亮）
  )
  hemisphereLight.position.set(0, 1, 0) // 从上方照射
  scene.add(hemisphereLight)

  // 💡 补充微弱环境光，确保模型无死角黑暗
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.3)
  scene.add(ambientLight)

  // 🔦 保留一个较弱的方向光，增加一点立体感（但不抢戏）
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5)
  directionalLight.position.set(2, 3, 2)
  directionalLight.castShadow = false // 不需要阴影
  scene.add(directionalLight)

  // 相机
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 0, 5)

  // 渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  container.value.appendChild(renderer.domElement)

  // 控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.rotateSpeed = 0.8
  controls.minDistance = 1.5
  controls.maxDistance = 15
  controls.enablePan = false // 可选：禁用平移，聚焦模型

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
  controls.update()
  renderer.render(scene, camera)
}

function preloadModel(modelName) {
  if (modelCache[modelName]) return

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

async function loadModel(modelName, retryCount = 0) {
  if (isLoading) return Promise.resolve()
  isLoading = true
  emit('loading', true)
  emit('progress', 0)

  if (modelCache[modelName]) {
    switchToModel(modelCache[modelName], modelName)
    isLoading = false
    emit('loading', false)
    return Promise.resolve()
  }

  const loader = new GLTFLoader()
  const oldModel = currentModel

  return new Promise((resolve, reject) => {
    loader.load(
      `/model/${modelName}`,
      (gltf) => {
        modelCache[modelName] = gltf.scene.clone()
        switchToModel(modelCache[modelName], modelName)
        isLoading = false
        emit('loading', false)
        resolve()
      },
      (xhr) => {
        if (xhr.lengthComputable) {
          const percent = (xhr.loaded / xhr.total) * 100
          emit('progress', percent)
        }
      },
      (error) => {
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
    )
  })
}

function switchToModel(modelScene, modelName) {
  if (currentModel) {
    scene.remove(currentModel)
  }
  currentModel = modelScene
  currentModel.userData.modelName = modelName
  scene.add(currentModel)
  controls.reset()
}

window.addEventListener('dblclick', () => {
  const currentName = currentModel?.userData.modelName || 'hunyuan01.glb'
  const nextModel = currentName === 'hunyuan01.glb' ? 'hunyuan02.glb' : 'hunyuan01.glb'

  if (modelCache[nextModel]) {
    switchToModel(modelCache[nextModel], nextModel)
  } else {
    loadModel(nextModel)
  }
})
</script>