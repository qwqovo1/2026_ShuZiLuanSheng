<!-- src/App.vue -->
<template>
  <div id="app">
    <ModelViewer
      @loading="onLoading"
      @progress="onProgress"
    />
    <div v-if="isLoading" class="loading-overlay">
      <div class="loading-content">
        <div class="spinner"></div>
        <p>Loading 3D Model...</p>
        <div v-if="progress > 0" class="progress-bar">
          <div
            class="progress-fill"
            :style="{ width: progress + '%' }"
          ></div>
        </div>
        <p v-if="progress > 0" class="progress-text">{{ Math.round(progress) }}%</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import ModelViewer from './components/ModelViewer.vue'

const isLoading = ref(true)
const progress = ref(0)

function onLoading(state) {
  isLoading.value = state
  if (!state) progress.value = 0
}

function onProgress(value) {
  progress.value = value
}
</script>

<style>
#app {
  width: 10 0vw;
  height: 100vh;
  margin: 0;
  overflow: hidden;
  position: relative;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.85);
  color: white;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  z-index: 1000;
}

.loading-content {
  text-align: center;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-top: 4px solid #4fc3f7;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 16px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.progress-bar {
  width: 200px;
  height: 6px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 3px;
  margin: 12px auto;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: #4fc3f7;
  transition: width 0.2s ease;
}

.progress-text {
  font-size: 0.9em;
  color: #bbb;
  margin-top: 4px;
}
</style>