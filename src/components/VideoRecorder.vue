<template>
  <div class="video-recorder">
    <!-- 视频显示区域 -->
    <div class="video-container">
      <video ref="liveVideo" class="video-element" autoplay muted playsinline v-show="recording"></video>
      <canvas ref="videoCanvas" class="video-canvas" v-show="!recording"></canvas>
      <div class="status-text">{{ statusText }}</div>
      <div class="progress-bar" v-show="progress > 0" :style="{ width: `${progress * 100}%` }"></div>
    </div>

    <!-- 控制按钮 -->
    <div class="controls">
      <button @click="toggleMode" class="control-btn">
        {{ recording ? '开始回放' : '继续录制' }}
      </button>
      <button @click="togglePause" class="control-btn" :disabled="recording || frameBuffer.length === 0">
        {{ paused ? '播放' : '暂停' }}
      </button>
      <button @click="cycleSpeed" class="control-btn" :disabled="recording || frameBuffer.length === 0">
        {{ playbackSpeed }}x
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

// 组件状态
const recording = ref(true)
const playing = ref(false)
const paused = ref(false)
const playbackSpeed = ref(0.25)
const statusText = ref('Record')
const frameBuffer = ref<ImageData[]>([])
const bufferSize = ref(5 * 30)
const currentPlaybackIndex = ref(0)
const playbackFrames = ref<ImageData[]>([])
const animationFrameId = ref<number | null>(null)
const playbackTimeoutId = ref<number | null>(null)

// DOM引用
const liveVideo = ref<HTMLVideoElement | null>(null)
const videoCanvas = ref<HTMLCanvasElement | null>(null)
const videoStream = ref<MediaStream | null>(null)
const canvasContext = ref<CanvasRenderingContext2D | null>(null)
const progress = ref<number>(-1)

// 切换播放速度
const speeds = [0.1, 0.25, 0.5, 1, 2]
const cycleSpeed = () => {
  const currentIndex = speeds.indexOf(playbackSpeed.value)
  const nextIndex = (currentIndex + 1) % speeds.length
  playbackSpeed.value = speeds[nextIndex]
  adjustSpeed(playbackSpeed.value)
}

// 初始化视频
onMounted(async () => {
  try {
    // 调用后置摄像头
    const stream = await navigator.mediaDevices.getUserMedia({
      video: {
        facingMode: { exact: "environment" }  // environment表示后置摄像头，user表示前置摄像头
      }
    })
    videoStream.value = stream

    if (liveVideo.value) {
      liveVideo.value.srcObject = stream
    }

    // 初始化canvas上下文
    if (videoCanvas.value) {
      const ctx = videoCanvas.value.getContext('2d')
      if (ctx) {
        canvasContext.value = ctx
      }
    }

    // 开始捕获帧
    startCapturingFrames()
  } catch (err) {
    console.error('无法访问摄像头:', err)
    alert('无法访问摄像头，请确保已授予摄像头权限')
  }
})

// 开始捕获帧
const startCapturingFrames = () => {
  if (!liveVideo.value || !videoCanvas.value || !canvasContext.value) return

  const captureFrame = () => {

    if (!videoCanvas.value) {
      return;
    }

    if (recording.value && !playing.value && liveVideo.value) {
      // 设置canvas尺寸与视频匹配
      if (videoCanvas.value) {
        videoCanvas.value.width = liveVideo.value.videoWidth
        videoCanvas.value.height = liveVideo.value.videoHeight
      }

      // 绘制当前帧到canvas
      canvasContext.value?.drawImage(
        liveVideo.value,
        0, 0,
        videoCanvas.value.width,
        videoCanvas.value.height
      )

      // 将当前帧存入缓冲区
      if (videoCanvas.value.width > 0 && videoCanvas.value.height > 0) {
        const frameData = canvasContext.value?.getImageData(
          0, 0,
          videoCanvas.value.width,
          videoCanvas.value.height
        )

        if (frameData) {
          frameBuffer.value.push(frameData)

          // 保持缓冲区大小
          if (frameBuffer.value.length > bufferSize.value) {
            frameBuffer.value.shift()
          }
        }
      }

      statusText.value = 'Recording'
    }

    animationFrameId.value = requestAnimationFrame(captureFrame)
  }

  animationFrameId.value = requestAnimationFrame(captureFrame)
}

// 切换录制/回放模式
const toggleMode = () => {
  recording.value = !recording.value
  playing.value = !recording.value
  paused.value = false

  if (recording.value) {
    progress.value = -1
  }

  if (playing.value) {
    startPlayback()
  } else {
    if (playbackTimeoutId.value) {
      clearTimeout(playbackTimeoutId.value)
      playbackTimeoutId.value = null
    }
  }
}

// 开始回放
const startPlayback = () => {
  if (frameBuffer.value.length === 0) return

  playbackFrames.value = [...frameBuffer.value]
  currentPlaybackIndex.value = 0
  playNextFrame()
  statusText.value = 'Recalling'
}

// 播放下一帧
const playNextFrame = () => {
  if (!videoCanvas.value || !canvasContext.value || !playing.value || paused.value) return

  if (currentPlaybackIndex.value >= playbackFrames.value.length) {
    // 循环播放
    currentPlaybackIndex.value = 0
  }

  progress.value = currentPlaybackIndex.value / playbackFrames.value.length

  const frame = playbackFrames.value[currentPlaybackIndex.value]
  canvasContext.value.putImageData(frame, 0, 0)

  currentPlaybackIndex.value++

  // 计算基于播放速度的延迟
  const delay = Math.max(16, 1000 / (30 * playbackSpeed.value)) // 假设30fps

  playbackTimeoutId.value = setTimeout(playNextFrame, delay) as unknown as number
}

// 切换暂停/播放状态
const togglePause = () => {
  paused.value = !paused.value
  if (!paused.value && playing.value) {
    playNextFrame()
  }
}

// 调整播放速度
const adjustSpeed = (speedFactor: number) => {
  playbackSpeed.value = speedFactor
  if (playing.value && !paused.value) {
    if (playbackTimeoutId.value) {
      clearTimeout(playbackTimeoutId.value)
    }
    playNextFrame()
  }
}

// 清理资源
onUnmounted(() => {
  if (animationFrameId.value) {
    cancelAnimationFrame(animationFrameId.value)
  }
  if (playbackTimeoutId.value) {
    clearTimeout(playbackTimeoutId.value)
  }
  if (videoStream.value) {
    videoStream.value.getTracks().forEach(track => track.stop())
  }
})
</script>

<style scoped>
.video-recorder {
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 20px;
  text-align: center;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
  background-color: #000;
  color: white;
}

h1 {
  margin-bottom: 20px;
  color: white;
}

.video-container {
  position: relative;
  flex-grow: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #000;
  overflow: hidden;
  margin-bottom: 20px;
}

.video-element,
.video-canvas {
  /* position: absolute; */
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  background-color: #000;
}

.status-text {
  position: absolute;
  top: 10px;
  left: 10px;
  color: white;
  font-size: 24px;
  font-weight: bold;
  text-shadow: 1px 1px 2px black;
}

.progress-bar {
  position: absolute;
  bottom: 0;
  left: 0;
  height: 2px;
  background-color: white;
  transition: width 0.1s linear;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 20px;
  padding-bottom: 20px;
}

.control-btn {
  padding: 8px 16px;
  font-size: 16px;
  cursor: pointer;
  background-color: black;
  color: white;
  border: 1px solid white;
  border-radius: 4px;
  transition: all 0.3s;
}

.control-btn:hover {
  background-color: #333;
}

.control-btn:disabled {
  background-color: #222;
  color: #888;
  border-color: #888;
  cursor: not-allowed;
}
</style>