<template>
  <div class="min-h-screen">
    <!-- 首页：字帖列表 -->
    <div v-if="!currentCalligraphy" class="container mx-auto px-4 py-8">
      <header class="text-center mb-12">
        <h1 class="text-4xl font-bold text-gray-800 mb-2">书法练习</h1>
        <p class="text-gray-500">选择字帖，开始临摹</p>
      </header>

      <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-6">
        <div
          v-for="item in calligraphyList"
          :key="item.id"
          @click="selectCalligraphy(item)"
          class="bg-white rounded-lg shadow-md overflow-hidden cursor-pointer hover:shadow-xl transition-shadow duration-300"
        >
          <div class="aspect-square bg-gray-100 flex items-center justify-center">
            <img 
              :src="item.image" 
              :alt="item.name"
              class="w-full h-full object-contain p-4"
              @error="handleImageError"
            />
          </div>
          <div class="p-4">
            <h3 class="font-bold text-lg text-gray-800">{{ item.name }}</h3>
            <p class="text-sm text-gray-500">{{ item.dynasty }} · {{ item.author }}</p>
          </div>
        </div>
      </div>
    </div>

    <!-- 练习页面 -->
    <div v-else class="h-screen flex flex-col">
      <!-- 工具栏 -->
      <div class="bg-white shadow-sm px-4 py-3 flex items-center justify-between">
        <button 
          @click="goBack"
          class="flex items-center gap-2 text-gray-600 hover:text-gray-800"
        >
          ← 返回
        </button>
        
        <div class="flex items-center gap-4">
          <!-- 笔画粗细 -->
          <div class="flex items-center gap-2">
            <span class="text-sm text-gray-500">粗细:</span>
            <button
              v-for="size in brushSizes"
              :key="size.value"
              @click="brushSize = size.value"
              :class="[
                'w-8 h-8 rounded-full flex items-center justify-center',
                brushSize === size.value ? 'bg-gray-800 text-white' : 'bg-gray-100'
              ]"
            >
              <span 
                class="rounded-full bg-current" 
                :style="{ width: size.preview + 'px', height: size.preview + 'px' }"
              ></span>
            </button>
          </div>

          <!-- 颜色选择 -->
          <div class="flex items-center gap-2">
            <span class="text-sm text-gray-500">颜色:</span>
            <button
              v-for="color in colors"
              :key="color.value"
              @click="inkColor = color.value"
              :class="[
                'w-8 h-8 rounded-full border-2',
                inkColor === color.value ? 'border-gray-800' : 'border-gray-200'
              ]"
              :style="{ backgroundColor: color.value }"
            ></button>
          </div>

          <!-- 字号调整 -->
          <div class="flex items-center gap-2">
            <button 
              @click="scale -= 0.1"
              class="w-8 h-8 rounded bg-gray-100 hover:bg-gray-200"
            >-</button>
            <span class="text-sm text-gray-500 w-12 text-center">{{ Math.round(scale * 100) }}%</span>
            <button 
              @click="scale += 0.1"
              class="w-8 h-8 rounded bg-gray-100 hover:bg-gray-200"
            >+</button>
          </div>

          <!-- 操作按钮 -->
          <button 
            @click="clearCanvas"
            class="px-4 py-2 bg-red-50 text-red-600 rounded hover:bg-red-100"
          >
            擦除
          </button>
          <button 
            @click="saveImage"
            class="px-4 py-2 bg-gray-800 text-white rounded hover:bg-gray-700"
          >
            保存
          </button>
        </div>
      </div>

      <!-- 临摹区域 -->
      <div class="flex-1 overflow-auto bg-gray-100 p-4 flex items-center justify-center">
        <div 
          class="relative bg-white shadow-lg"
          :style="{ 
            width: containerWidth + 'px', 
            height: containerHeight + 'px',
            transform: `scale(${scale})`,
            transformOrigin: 'center center'
          }"
        >
          <!-- 字帖层 -->
          <img 
            :src="currentCalligraphy.image"
            class="absolute inset-0 w-full h-full object-contain"
            @error="handleImageError"
          />
          
          <!-- 书写层 -->
          <canvas
            ref="canvasRef"
            class="absolute inset-0 w-full h-full cursor-crosshair touch-none"
            @mousedown="startDraw"
            @mousemove="draw"
            @mouseup="stopDraw"
            @mouseleave="stopDraw"
            @touchstart.prevent="startDraw"
            @touchmove.prevent="draw"
            @touchend.prevent="stopDraw"
          ></canvas>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import { calligraphyList } from './data/calligraphy.js'

const currentCalligraphy = ref(null)
const canvasRef = ref(null)
let ctx = null

// 画笔设置
const brushSize = ref(8)
const brushSizes = [
  { value: 4, preview: 4 },
  { value: 8, preview: 8 },
  { value: 16, preview: 16 }
]

const inkColor = ref('#1a1a1a')
const colors = [
  { value: '#1a1a1a', name: '墨黑' },
  { value: '#c41e3a', name: '朱红' },
  { value: '#1e4d8c', name: '靛蓝' }
]

// 缩放
const scale = ref(1)

// 画布尺寸
const containerWidth = ref(600)
const containerHeight = ref(600)

// 绘图状态
const isDrawing = ref(false)
let lastX = 0
let lastY = 0

// 选择字帖
const selectCalligraphy = async (item) => {
  currentCalligraphy.value = item
  await nextTick()
  initCanvas()
}

// 返回首页
const goBack = () => {
  currentCalligraphy.value = null
}

// 初始化画布
const initCanvas = () => {
  const canvas = canvasRef.value
  if (!canvas) return
  
  ctx = canvas.getContext('2d')
  
  // 设置画布尺寸
  const rect = canvas.getBoundingClientRect()
  canvas.width = rect.width
  canvas.height = rect.height
  
  // 设置线条样式
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'
}

// 开始绘制
const startDraw = (e) => {
  isDrawing.value = true
  const { x, y } = getPosition(e)
  lastX = x
  lastY = y
}

// 绘制中
const draw = (e) => {
  if (!isDrawing.value || !ctx) return
  
  const { x, y } = getPosition(e)
  
  ctx.strokeStyle = inkColor.value
  ctx.lineWidth = brushSize.value
  
  ctx.beginPath()
  ctx.moveTo(lastX, lastY)
  ctx.lineTo(x, y)
  ctx.stroke()
  
  lastX = x
  lastY = y
}

// 停止绘制
const stopDraw = () => {
  isDrawing.value = false
}

// 获取坐标
const getPosition = (e) => {
  const canvas = canvasRef.value
  const rect = canvas.getBoundingClientRect()
  
  let clientX, clientY
  if (e.touches) {
    clientX = e.touches[0].clientX
    clientY = e.touches[0].clientY
  } else {
    clientX = e.clientX
    clientY = e.clientY
  }
  
  return {
    x: clientX - rect.left,
    y: clientY - rect.top
  }
}

// 擦除
const clearCanvas = () => {
  if (!ctx || !canvasRef.value) return
  ctx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height)
}

// 保存图片
const saveImage = () => {
  if (!canvasRef.value || !currentCalligraphy.value) return
  
  // 创建合并画布
  const mergeCanvas = document.createElement('canvas')
  mergeCanvas.width = containerWidth.value
  mergeCanvas.height = containerHeight.value
  const mergeCtx = mergeCanvas.getContext('2d')
  
  // 白色背景
  mergeCtx.fillStyle = '#ffffff'
  mergeCtx.fillRect(0, 0, mergeCanvas.width, mergeCanvas.height)
  
  // 绘制字帖
  const img = new Image()
  img.crossOrigin = 'anonymous'
  img.onload = () => {
    mergeCtx.drawImage(img, 0, 0, mergeCanvas.width, mergeCanvas.height)
    
    // 绘制用户书写
    mergeCtx.drawImage(canvasRef.value, 0, 0)
    
    // 下载
    const link = document.createElement('a')
    link.download = `书法练习-${currentCalligraphy.value.name}.png`
    link.href = mergeCanvas.toDataURL('image/png')
    link.click()
  }
  img.src = currentCalligraphy.value.image
}

// 图片加载失败处理
const handleImageError = (e) => {
  e.target.style.display = 'none'
}

onMounted(() => {
  initCanvas()
})
</script>
