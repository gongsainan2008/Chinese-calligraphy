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
      <div class="bg-white shadow-sm px-4 py-3 flex items-center justify-between flex-wrap gap-2">
        <button 
          @click="goBack"
          class="flex items-center gap-2 text-gray-600 hover:text-gray-800"
        >
          ← 返回
        </button>
        
        <div class="flex items-center gap-4 flex-wrap">
          <!-- 笔画粗细 -->
          <div class="flex items-center gap-2">
            <span class="text-sm text-gray-500">笔锋:</span>
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
            <span class="text-sm text-gray-500">墨色:</span>
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
              @click="scale = Math.max(0.5, scale - 0.1)"
              class="w-8 h-8 rounded bg-gray-100 hover:bg-gray-200"
            >-</button>
            <span class="text-sm text-gray-500 w-12 text-center">{{ Math.round(scale * 100) }}%</span>
            <button 
              @click="scale = Math.min(1.5, scale + 0.1)"
              class="w-8 h-8 rounded bg-gray-100 hover:bg-gray-200"
            >+</button>
          </div>

          <!-- 米字格开关 -->
          <button 
            @click="showGrid = !showGrid"
            :class="[
              'px-4 py-2 rounded',
              showGrid ? 'bg-gray-800 text-white' : 'bg-gray-100 text-gray-600'
            ]"
          >
            米字格
          </button>

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
      <div class="flex-1 overflow-auto bg-[#e8e4dc] p-4 flex items-center justify-center">
        <div 
          class="relative shadow-2xl"
          :style="{ 
            width: containerWidth + 'px', 
            height: containerHeight + 'px',
            transform: `scale(${scale})`,
            transformOrigin: 'center center'
          }"
        >
          <!-- 字帖层 -->
          <div 
            class="absolute inset-0 bg-white"
            :style="{ backgroundImage: `url(${currentCalligraphy.image})`, backgroundSize: 'contain', backgroundPosition: 'center', backgroundRepeat: 'no-repeat' }"
          ></div>
          
          <!-- 米字格层 -->
          <div v-if="showGrid" class="absolute inset-0 pointer-events-none">
            <svg class="w-full h-full" viewBox="0 0 600 600" preserveAspectRatio="none">
              <!-- 外框 -->
              <rect x="1" y="1" width="598" height="598" fill="none" stroke="#c4b5a0" stroke-width="2"/>
              <!-- 横中线 -->
              <line x1="1" y1="300" x2="599" y2="300" stroke="#c4b5a0" stroke-width="1" stroke-dasharray="8,4"/>
              <!-- 竖中线 -->
              <line x1="300" y1="1" x2="300" y2="599" stroke="#c4b5a0" stroke-width="1" stroke-dasharray="8,4"/>
              <!-- 左斜线 -->
              <line x1="1" y1="1" x2="599" y2="599" stroke="#c4b5a0" stroke-width="1" stroke-dasharray="8,4"/>
              <!-- 右斜线 -->
              <line x1="599" y1="1" x2="1" y2="599" stroke="#c4b5a0" stroke-width="1" stroke-dasharray="8,4"/>
            </svg>
          </div>
          
          <!-- 书写层（半透明练习纸效果） -->
          <canvas
            ref="canvasRef"
            class="absolute inset-0 w-full h-full cursor-crosshair touch-none"
            style="background-color: rgba(255, 255, 255, 0.25);"
            @mousedown="startDraw"
            @mousemove="draw"
            @mouseup="stopDraw"
            @mouseleave="stopDraw"
            @contextmenu.prevent="stopDraw"
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

// 画笔设置（硬笔书法）
const brushSize = ref(6)
const brushSizes = [
  { value: 3, preview: 3 },
  { value: 6, preview: 6 },
  { value: 10, preview: 10 }
]

const inkColor = ref('#1a1a1a')
const colors = [
  { value: '#1a1a1a', name: '墨黑' },
  { value: '#8B0000', name: '朱砂' },
  { value: '#1e4d8c', name: '靛蓝' }
]

// 缩放
const scale = ref(1)

// 画布尺寸
const containerWidth = ref(600)
const containerHeight = ref(600)

// 米字格显示
const showGrid = ref(true)

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
  canvas.width = rect.width * 2  // 2x 分辨率，更清晰
  canvas.height = rect.height * 2
  ctx.scale(2, 2)
  
  // 默认不设置任何背景色，保持完全透明
}

// 开始绘制（点击开始）
const startDraw = (e) => {
  // 阻止默认事件
  e.preventDefault()
  // 左键或触摸开始
  if (e.type === 'mousedown' && e.button !== 0) return
  
  isDrawing.value = true
  const { x, y } = getPosition(e)
  lastX = x
  lastY = y
}

// 绘制中 - 硬笔书法效果
const draw = (e) => {
  if (!isDrawing.value || !ctx) return
  
  const { x, y } = getPosition(e)
  
  // 硬笔书法：均匀线条，无粗细变化
  ctx.strokeStyle = inkColor.value
  ctx.lineWidth = brushSize.value
  ctx.lineCap = 'round'
  ctx.lineJoin = 'round'
  
  ctx.beginPath()
  ctx.moveTo(lastX, lastY)
  ctx.lineTo(x, y)
  ctx.stroke()
  
  lastX = x
  lastY = y
}

// 停止绘制（右键或松手）
const stopDraw = (e) => {
  // 右键结束
  if (e && e.type === 'contextmenu') {
    e.preventDefault()
  }
  isDrawing.value = false
}

// 获取坐标
const getPosition = (e) => {
  const canvas = canvasRef.value
  const rect = canvas.getBoundingClientRect()
  
  let clientX, clientY
  
  if (e.touches && e.touches.length > 0) {
    clientX = e.touches[0].clientX
    clientY = e.touches[0].clientY
  } else {
    clientX = e.clientX
    clientY = e.clientY
  }
  
  // 缩放影响
  const scaleX = canvas.width / 2 / rect.width
  const scaleY = canvas.height / 2 / rect.height
  
  return {
    x: (clientX - rect.left) * scaleX,
    y: (clientY - rect.top) * scaleY
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
  const w = containerWidth.value
  const h = containerHeight.value
  mergeCanvas.width = w * 2
  mergeCanvas.height = h * 2
  const mergeCtx = mergeCanvas.getContext('2d')
  mergeCtx.scale(2, 2)
  
  // 白色背景（仿宣纸）
  mergeCtx.fillStyle = '#faf8f5'
  mergeCtx.fillRect(0, 0, w, h)
  
  // 绘制字帖
  const img = new Image()
  img.crossOrigin = 'anonymous'
  img.onload = () => {
    mergeCtx.drawImage(img, 0, 0, w, h)
    
    // 绘制半透明背景层
    mergeCtx.fillStyle = 'rgba(255, 255, 255, 0.25)'
    mergeCtx.fillRect(0, 0, w, h)
    
    // 绘制用户书写
    mergeCtx.drawImage(canvasRef.value, 0, 0, w, h)
    
    // 如果显示米字格，绘制格子
    if (showGrid.value) {
      mergeCtx.strokeStyle = '#c4b5a0'
      mergeCtx.lineWidth = 1
      // 外框
      mergeCtx.strokeRect(1, 1, w - 2, h - 2)
      // 横中线
      mergeCtx.setLineDash([8, 4])
      mergeCtx.beginPath()
      mergeCtx.moveTo(1, h / 2)
      mergeCtx.lineTo(w - 1, h / 2)
      mergeCtx.stroke()
      // 竖中线
      mergeCtx.beginPath()
      mergeCtx.moveTo(w / 2, 1)
      mergeCtx.lineTo(w / 2, h - 1)
      mergeCtx.stroke()
      // 对角线
      mergeCtx.beginPath()
      mergeCtx.moveTo(1, 1)
      mergeCtx.lineTo(w - 1, h - 1)
      mergeCtx.stroke()
      mergeCtx.beginPath()
      mergeCtx.moveTo(w - 1, 1)
      mergeCtx.lineTo(1, h - 1)
      mergeCtx.stroke()
      mergeCtx.setLineDash([])
    }
    
    // 下载
    const link = document.createElement('a')
    link.download = `书法练习-${currentCalligraphy.value.name}.png`
    link.href = mergeCanvas.toDataURL('image/png')
    link.click()
  }
  // 使用 SVG 作为字帖源
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

<style scoped>
/* 宣纸纹理效果 */
.bg-\[\#e8e4dc\] {
  background-color: #e8e4dc;
  background-image: 
    radial-gradient(ellipse at center, rgba(255,255,255,0.1) 0%, transparent 70%),
    url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.65' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)' opacity='0.05'/%3E%3C/svg%3E");
}
</style>
