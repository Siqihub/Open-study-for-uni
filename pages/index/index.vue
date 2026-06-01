<template>
  <view class="container">
    <view class="header">
      <text class="title">照片水印工具</text>
      <text class="subtitle">支持 H5 和 App 双端</text>
    </view>

    <view class="card">
      <view class="section-title">图片选择</view>
      <button class="btn-primary" @click="chooseImages">
        <text class="btn-icon">+</text>
        <text>选择图片（可多选）</text>
      </button>
      <view class="thumb-list" v-if="images.length">
        <view
          v-for="(item, index) in images"
          :key="index"
          class="thumb-wrap"
          @click="onImageClick(index)"
          @longpress="onImageLongPress(index)"
        >
          <image :src="item" class="thumb" mode="aspectFill" />
          <view class="thumb-overlay" v-if="isMultiSelectMode"></view>
          <view class="thumb-index">{{ index + 1 }}</view>
          <view v-if="isMultiSelectMode" class="thumb-check" :class="{ active: selectedIndexes.includes(index) }">
            <text v-if="selectedIndexes.includes(index)">✓</text>
          </view>
          <view v-if="isMultiSelectMode && !selectedIndexes.includes(index)" class="thumb-check-placeholder"></view>
        </view>
      </view>
      <view class="empty" v-else>点击上方按钮选择需要加水印的照片</view>
      <view class="delete-bar" v-if="isMultiSelectMode">
        <text class="delete-info">已选 {{ selectedIndexes.length }} 张</text>
        <view class="delete-actions">
          <button class="btn-cancel" @click="cancelMultiSelect">取消</button>
          <button class="btn-delete" @click="deleteSelected">删除选中</button>
        </view>
      </view>
    </view>

    <view class="card">
      <view class="section-title">预设配置</view>
      <view class="preset-row">
        <picker mode="selector" :range="presetNames" :value="presetIndex" @change="onPresetChange">
          <view class="picker-box">
            <text class="picker-label">当前预设：</text>
            <text class="picker-value">{{ presetNames[presetIndex] || '默认' }}</text>
            <text class="picker-arrow">▼</text>
          </view>
        </picker>
        <button class="btn-mini" @click="savePreset">保存为新预设</button>
      </view>
    </view>

    <view class="card">
      <view class="section-title">水印参数</view>
      <view class="form-item">
        <view class="label-row">
          <text class="label">地点</text>
          <view class="history-toggle" @click="showHistory = !showHistory">
            <text class="history-toggle-text">历史</text>
            <text class="history-toggle-icon" :class="{ open: showHistory }">▼</text>
          </view>
        </view>
        <input v-model="location" placeholder="请输入地点，如：三亚市吉阳区海南热带海洋学院" class="input" />
        <view class="history-dropdown" v-if="showHistory && historyLocations.length > 0">
          <view
            v-for="(loc, idx) in historyLocations"
            :key="idx"
            class="history-item"
            @click="selectHistoryLocation(loc)"
          >
            <text class="history-rank">{{ idx + 1 }}</text>
            <text class="history-text">{{ loc }}</text>
            <text class="history-freq" v-if="locationFreq[loc]">{{ locationFreq[loc] }}次</text>
          </view>
        </view>
      </view>

      <view class="form-item">
        <text class="label">基准时间</text>
        <picker mode="multiSelector" :range="dateTimeRange" :value="dateTimeIndex" @change="onDateTimeChange" @columnchange="onDateTimeColumnChange">
          <view class="picker-box">
            <text class="picker-value">{{ formatDateTime(dateTimeIndex) }}</text>
            <text class="picker-arrow">▼</text>
          </view>
        </picker>
      </view>

      <view class="form-item">
        <text class="label">水印大小: {{ fontScale }}</text>
        <slider :value="fontScale" @changing="onFontScaleChanging" @change="onFontScaleChange" min="20" max="120" show-value />
        <view class="preview-box">
          <text class="preview-label">预览效果：</text>
          <text class="preview-text" :style="{ fontSize: previewFontSize + 'px', fontFamily: fontNames[fontIndex] }">水印文字预览</text>
        </view>
      </view>

      <view class="form-item">
        <text class="label">字体</text>
        <picker mode="selector" :range="fontNames" :value="fontIndex" @change="onFontChange">
          <view class="picker-box">
            <text class="picker-value" :style="{ fontFamily: fontNames[fontIndex] }">{{ fontNames[fontIndex] }}</text>
            <text class="picker-arrow">▼</text>
          </view>
        </picker>
        <view class="font-preview-list">
          <view
            v-for="(font, idx) in fontNames"
            :key="idx"
            class="font-preview-item"
            :class="{ active: fontIndex === idx }"
            @click="fontIndex = idx"
            :style="{ fontFamily: font }"
          >
            {{ font }}
          </view>
        </view>
      </view>

      <view class="form-item switch-row">
        <view class="switch-info">
          <text class="label">时间随机偏移</text>
          <text class="switch-desc">在基准时间上增加 0~10 分钟随机偏移</text>
        </view>
        <switch :checked="randomTime" @change="e => randomTime = e.detail.value" color="#007aff" />
      </view>
    </view>

    <view class="card action-card">
      <button
        class="btn-process"
        :class="{ disabled: !images.length || processing }"
        :disabled="!images.length || processing"
        @click="batchWatermark"
      >
        <text v-if="!processing">批量处理并保存</text>
        <text v-else>处理中...</text>
      </button>
      <view class="platform-tip">
        <text v-if="isH5">💻 当前为 H5 模式，将触发浏览器下载</text>
        <text v-else>📱 当前为 App 模式，将保存到系统相册</text>
      </view>
    </view>

    <view class="card" v-if="previewImages.length">
      <view class="section-title">处理结果预览（双击可重新处理）</view>
      <view class="thumb-list">
        <view
          v-for="(item, index) in previewImages"
          :key="index"
          class="thumb-wrap"
          @click="handleImageClick(item, index)"
        >
          <image :src="item" class="thumb" mode="aspectFill" />
          <view class="thumb-index">{{ index + 1 }}</view>
        </view>
      </view>
      <view class="preview-actions">
        <button class="btn-save" @click="saveAllImages">💾 保存</button>
        <button class="btn-refresh" @click="refreshAllImages">🔄 刷新</button>
        <button class="btn-clear" @click="clearPreview">❌ 清除</button>
      </view>
    </view>

    <canvas
      canvas-id="watermarkCanvas"
      id="watermarkCanvas"
      :style="canvasStyle"
    />
  </view>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const STORAGE_KEY = 'watermark_presets_v1'
const HISTORY_KEY = 'watermark_history_v1'
const FREQ_KEY = 'watermark_freq_v1'
const DEFAULT_LOCATION = '三亚市吉阳区海南热带海洋学院'

const isH5 = ref(false)

const images = ref([])
const selectedIndexes = ref([])
const isMultiSelectMode = ref(false)
const location = ref('')
const randomTime = ref(false)
const processing = ref(false)
const resultImages = ref([])
const previewImages = ref([])
const showHistory = ref(false)
const lastClickTime = ref(0)
const lastClickIndex = ref(-1)

const fontScale = ref(45)
const fontNames = ['sans-serif', 'serif', 'monospace', 'Arial', 'Helvetica', 'Georgia', 'Verdana', 'Tahoma', 'Times New Roman', 'Courier New', 'Impact', 'Comic Sans MS']
const fontIndex = ref(0)

const canvasWidth = ref(1)
const canvasHeight = ref(1)
const canvasStyle = computed(() => {
  return `position: fixed; left: -9999px; top: -9999px; width: ${canvasWidth.value}px; height: ${canvasHeight.value}px;`
})

const previewFontSize = computed(() => {
  return Math.max(12, Math.min(24, fontScale.value / 3))
})

const presets = ref([])
const presetIndex = ref(0)
const presetNames = computed(() => {
  const names = presets.value.map(p => p.name)
  names.unshift('默认（不应用预设）')
  return names
})

const historyLocations = ref([])
const locationFreq = ref({})

const now = new Date()
const years = Array.from({ length: 10 }, (_, i) => (now.getFullYear() - 5 + i) + '年')
const months = Array.from({ length: 12 }, (_, i) => (i + 1) + '月')
const days = Array.from({ length: 31 }, (_, i) => (i + 1) + '日')
const hours = Array.from({ length: 24 }, (_, i) => i.toString().padStart(2, '0') + '时')
const minutes = Array.from({ length: 60 }, (_, i) => i.toString().padStart(2, '0') + '分')

const dateTimeRange = ref([years, months, days, hours, minutes])
const dateTimeIndex = ref([
  years.indexOf(now.getFullYear() + '年'),
  now.getMonth(),
  now.getDate() - 1,
  now.getHours(),
  now.getMinutes()
])

function formatDateTime(indexArr) {
  const y = years[indexArr[0]]
  const m = months[indexArr[1]]
  const d = days[indexArr[2]]
  const h = hours[indexArr[3]]
  const min = minutes[indexArr[4]]
  return `${y}${m}${d} ${h}${min}`
}

function getBaseDate() {
  const y = parseInt(years[dateTimeIndex.value[0]])
  const m = parseInt(months[dateTimeIndex.value[1]])
  const d = parseInt(days[dateTimeIndex.value[2]])
  const h = parseInt(hours[dateTimeIndex.value[3]])
  const min = parseInt(minutes[dateTimeIndex.value[4]])
  return new Date(y, m - 1, d, h, min, 0)
}

function onDateTimeChange(e) {
  dateTimeIndex.value = e.detail.value
}

function onDateTimeColumnChange(e) {
  const col = e.detail.column
  const val = e.detail.value
  dateTimeIndex.value[col] = val
}

function onFontScaleChanging(e) {
  fontScale.value = e.detail.value
}

function onFontScaleChange(e) {
  fontScale.value = e.detail.value
}

function onFontChange(e) {
  fontIndex.value = e.detail.value
}

function addToHistory(loc) {
  if (!loc) return
  
  if (!locationFreq.value[loc]) {
    locationFreq.value[loc] = 0
  }
  locationFreq.value[loc]++
  uni.setStorageSync(FREQ_KEY, JSON.stringify(locationFreq.value))
  
  const sorted = Object.entries(locationFreq.value)
    .sort((a, b) => b[1] - a[1])
    .map(([key]) => key)
    .slice(0, 10)
  
  historyLocations.value = sorted
  uni.setStorageSync(HISTORY_KEY, JSON.stringify(sorted))
}

function loadHistory() {
  try {
    const data = uni.getStorageSync(HISTORY_KEY)
    if (data) {
      historyLocations.value = JSON.parse(data)
    }
  } catch (e) {
    console.error('读取历史地址失败', e)
  }
  
  try {
    const freqData = uni.getStorageSync(FREQ_KEY)
    if (freqData) {
      locationFreq.value = JSON.parse(freqData)
    }
  } catch (e) {
    console.error('读取频率失败', e)
  }
  
  if (historyLocations.value.length === 0) {
    historyLocations.value = [DEFAULT_LOCATION]
    locationFreq.value[DEFAULT_LOCATION] = 1
  }
  location.value = historyLocations.value[0]
}

function selectHistoryLocation(loc) {
  location.value = loc
  showHistory.value = false
}

function loadPresets() {
  try {
    const data = uni.getStorageSync(STORAGE_KEY)
    if (data) {
      presets.value = JSON.parse(data)
    }
  } catch (e) {
    console.error('读取预设失败', e)
  }
}

function savePreset() {
  uni.showModal({
    title: '保存预设',
    editable: true,
    placeholderText: '请输入预设名称，如：上班打卡',
    success: (res) => {
      if (res.confirm && res.content && res.content.trim()) {
        const preset = {
          name: res.content.trim(),
          location: location.value,
          randomTime: randomTime.value,
          fontScale: fontScale.value,
          fontIndex: fontIndex.value,
          dateTimeIndex: [...dateTimeIndex.value]
        }
        presets.value.push(preset)
        uni.setStorageSync(STORAGE_KEY, JSON.stringify(presets.value))
        presetIndex.value = presets.value.length
        addToHistory(location.value)
        uni.showToast({ title: '保存成功', icon: 'success' })
      }
    }
  })
}

function onPresetChange(e) {
  const idx = e.detail.value
  presetIndex.value = idx
  if (idx === 0) {
    location.value = historyLocations.value[0] || DEFAULT_LOCATION
    randomTime.value = false
    fontScale.value = 45
    fontIndex.value = 0
    const n = new Date()
    dateTimeIndex.value = [
      years.indexOf(n.getFullYear() + '年'),
      n.getMonth(),
      n.getDate() - 1,
      n.getHours(),
      n.getMinutes()
    ]
  } else {
    const p = presets.value[idx - 1]
    if (p) {
      location.value = p.location || DEFAULT_LOCATION
      randomTime.value = !!p.randomTime
      fontScale.value = p.fontScale || 45
      fontIndex.value = p.fontIndex || 0
      if (p.dateTimeIndex && p.dateTimeIndex.length === 5) {
        dateTimeIndex.value = [...p.dateTimeIndex]
      }
    }
  }
}

function chooseImages() {
  uni.chooseImage({
    count: 9,
    sizeType: ['original'],
    sourceType: ['album'],
    success: (res) => {
      images.value = [...images.value, ...res.tempFilePaths]
      selectedIndexes.value = []
      isMultiSelectMode.value = false
    },
    fail: (err) => {
      console.error('选择图片失败', err)
    }
  })
}

function onImageClick(index) {
  if (isMultiSelectMode.value) {
    toggleSelect(index)
  } else {
    previewImage(index)
  }
}

function onImageLongPress(index) {
  if (!isMultiSelectMode.value) {
    isMultiSelectMode.value = true
    selectedIndexes.value = [index]
  }
}

function previewImage(index) {
  uni.previewImage({
    urls: images.value,
    current: images.value[index]
  })
}

function toggleSelect(index) {
  const i = selectedIndexes.value.indexOf(index)
  if (i > -1) {
    selectedIndexes.value.splice(i, 1)
  } else {
    selectedIndexes.value.push(index)
  }
}

function cancelMultiSelect() {
  isMultiSelectMode.value = false
  selectedIndexes.value = []
}

function deleteSelected() {
  uni.showModal({
    title: '删除图片',
    content: `确定要删除选中的 ${selectedIndexes.value.length} 张图片吗？`,
    confirmColor: '#f5576c',
    success: (res) => {
      if (res.confirm) {
        const sorted = [...selectedIndexes.value].sort((a, b) => b - a)
        sorted.forEach(idx => {
          images.value.splice(idx, 1)
        })
        selectedIndexes.value = []
        isMultiSelectMode.value = false
      }
    }
  })
}

function batchWatermark() {
  if (!images.value.length || processing.value) {
    return
  }

  previewImages.value = []
  processing.value = true
  const list = [...images.value]
  let current = 0

  function next() {
    if (current >= list.length) {
      processing.value = false
      uni.hideLoading()
      uni.showToast({ title: '预览生成完成，请确认后保存', icon: 'success' })
      return
    }

    uni.showLoading({
      title: `正在处理 ${current + 1}/${list.length}`,
      mask: true
    })

    processImage(list[current], current, (resultPath) => {
      if (resultPath) {
        previewImages.value.push(resultPath)
      }
      current++
      next()
    })
  }

  next()
}

function reprocessImage(index) {
  if (processing.value || index < 0 || index >= images.value.length) {
    return
  }

  uni.showLoading({
    title: `重新处理第 ${index + 1} 张`,
    mask: true
  })

  processImage(images.value[index], index, (resultPath) => {
    uni.hideLoading()
    if (resultPath) {
      previewImages.value[index] = resultPath
      uni.showToast({ title: '重新处理完成', icon: 'success' })
    }
  })
}

function refreshAllImages() {
  if (!images.value.length || processing.value) {
    return
  }

  // 删除之前的预览缓存文件
  previewImages.value.forEach(path => {
    if (path && path.startsWith('file://')) {
      uni.getSavedFileInfo({
        filePath: path,
        success: () => {
          uni.removeSavedFile({ filePath: path })
        }
      })
    }
  })

  previewImages.value = []
  processing.value = true
  const list = [...images.value]
  let current = 0

  function next() {
    if (current >= list.length) {
      processing.value = false
      uni.hideLoading()
      uni.showToast({ title: '全部刷新完成', icon: 'success' })
      return
    }

    uni.showLoading({
      title: `正在刷新 ${current + 1}/${list.length}`,
      mask: true
    })

    processImage(list[current], current, (resultPath) => {
      if (resultPath) {
        previewImages.value.push(resultPath)
      }
      current++
      next()
    })
  }

  next()
}

function saveAllImages() {
  if (!previewImages.value.length) {
    uni.showToast({ title: '没有可保存的图片', icon: 'none' })
    return
  }

  // 清理之前的 resultImages 缓存
  resultImages.value.forEach(path => {
    if (path && path.startsWith('file://')) {
      uni.getSavedFileInfo({
        filePath: path,
        success: () => {
          uni.removeSavedFile({ filePath: path })
        }
      })
    }
  })

  resultImages.value = [...previewImages.value]
  let savedCount = 0

  uni.showLoading({
    title: '正在保存...',
    mask: true
  })

  previewImages.value.forEach((path, index) => {
    saveFile(path, index, () => {
      savedCount++
      if (savedCount >= previewImages.value.length) {
        uni.hideLoading()
        uni.showToast({ title: '全部保存完成', icon: 'success' })
        
        // 保存完成后释放资源
        releaseResources()
      }
    })
  })
}

function releaseResources() {
  // 删除预览缓存文件
  previewImages.value.forEach(path => {
    if (path && path.startsWith('file://')) {
      uni.getSavedFileInfo({
        filePath: path,
        success: () => {
          uni.removeSavedFile({ filePath: path })
        }
      })
    }
  })
  
  // 清空数据
  previewImages.value = []
  images.value = []
  selectedIndexes.value = []
  isMultiSelectMode.value = false
  
  uni.showToast({ title: '已释放空间', icon: 'none' })
}

function clearPreview() {
  previewImages.value = []
  uni.showToast({ title: '预览已清除', icon: 'none' })
}

function processImage(src, index, callback) {
  uni.getImageInfo({
    src,
    success: (imgInfo) => {
      let w = imgInfo.width
      let h = imgInfo.height
      
      const MAX_SIZE = 1920
      if (w > MAX_SIZE || h > MAX_SIZE) {
        if (w > h) {
          h = Math.round(h * MAX_SIZE / w)
          w = MAX_SIZE
        } else {
          w = Math.round(w * MAX_SIZE / h)
          h = MAX_SIZE
        }
      }

      canvasWidth.value = w
      canvasHeight.value = h

      const ctx = uni.createCanvasContext('watermarkCanvas')

      ctx.drawImage(src, 0, 0, w, h)

      const fontSize = Math.max(16, Math.round(w * (fontScale.value / 1000)))
      const margin = Math.round(w * 0.04)
      const lineHeight = fontSize * 1.4

      let baseDate = getBaseDate()
      if (randomTime.value) {
        const addMin = Math.floor(Math.random() * 11)
        const addSec = Math.floor(Math.random() * 60)
        baseDate = new Date(baseDate.getTime() + addMin * 60000 + addSec * 1000)
      }

      const timeStr = formatDate(baseDate)
      const locStr = location.value || ''

      const fontName = fontNames[fontIndex.value] || 'sans-serif'
      
      ctx.setFontSize(fontSize)
      ctx.setTextAlign('right')
      ctx.setTextBaseline('bottom')

      const x = w - margin
      const y1 = h - margin - lineHeight
      const y2 = h - margin

      if (locStr) {
        drawStrokeText(ctx, locStr, x, y1, fontSize, fontName)
      }
      drawStrokeText(ctx, timeStr, x, y2, fontSize, fontName)

      ctx.draw(false, () => {
        uni.canvasToTempFilePath({
          canvasId: 'watermarkCanvas',
          x: 0,
          y: 0,
          width: w,
          height: h,
          destWidth: w,
          destHeight: h,
          fileType: 'jpg',
          quality: 0.9,
          success: (res) => {
            addToHistory(location.value)
            saveFile(res.tempFilePath, index, () => {
              if (callback) callback(res.tempFilePath)
            })
          },
          fail: (err) => {
            console.error('canvas导出失败', err)
            uni.showToast({ title: '导出失败', icon: 'none' })
            if (callback) callback(null)
          }
        })
      })
    },
    fail: (err) => {
      console.error('获取图片信息失败', err)
      uni.showToast({ title: '读取图片失败', icon: 'none' })
      if (callback) callback(null)
    }
  })
}

function saveFile(filePath, index, callback) {
  const timestamp = Date.now()
  const filename = `watermark_${timestamp}_${index + 1}.jpg`

  // #ifdef APP-PLUS
  uni.saveImageToPhotosAlbum({
    filePath: filePath,
    success: () => {
      if (callback) callback()
    },
    fail: (err) => {
      console.error('保存相册失败', err)
      uni.showToast({ title: '保存相册失败', icon: 'none' })
      if (callback) callback()
    }
  })
  // #endif

  // #ifdef H5
  try {
    if (typeof document !== 'undefined') {
      const link = document.createElement('a')
      link.href = filePath
      link.download = filename
      link.style.display = 'none'
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    }
  } catch (e) {
    console.error('H5保存失败', e)
  }
  setTimeout(() => {
    if (callback) callback()
  }, 100)
  // #endif
}

function handleImageClick(path, index) {
  const currentTime = Date.now()
  const timeDiff = currentTime - lastClickTime.value

  if (timeDiff < 300 && lastClickIndex.value === index) {
    lastClickTime.value = 0
    lastClickIndex.value = -1
    reprocessImage(index)
  } else {
    lastClickTime.value = currentTime
    lastClickIndex.value = index
    setTimeout(() => {
      if (lastClickTime.value === currentTime) {
        previewResult(path)
      }
    }, 300)
  }
}

function previewResult(path) {
  uni.previewImage({
    urls: previewImages.value,
    current: path,
    success: () => {
      console.log('预览成功')
    },
    fail: (err) => {
      console.error('预览失败', err)
      uni.showToast({ title: '预览失败，请重试', icon: 'none' })
    }
  })
}

function drawStrokeText(ctx, text, x, y, fontSize, fontName) {
  ctx.font = `${fontSize}px ${fontName}`
  
  ctx.setFillStyle('#000000')
  ctx.setStrokeStyle('#000000')
  ctx.setLineWidth(fontSize * 0.15)
  ctx.strokeText(text, x, y)
  ctx.fillText(text, x, y)

  ctx.setFillStyle('#ffffff')
  ctx.fillText(text, x, y)
}

function formatDate(date) {
  const y = date.getFullYear()
  const m = (date.getMonth() + 1).toString().padStart(2, '0')
  const d = date.getDate().toString().padStart(2, '0')
  const h = date.getHours().toString().padStart(2, '0')
  const min = date.getMinutes().toString().padStart(2, '0')
  const s = date.getSeconds().toString().padStart(2, '0')
  return `${y}-${m}-${d} ${h}:${min}:${s}`
}

onMounted(() => {
  loadPresets()
  loadHistory()
  // #ifdef H5
  isH5.value = true
  // #endif
})
</script>

<style scoped>
.container {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 30rpx;
  padding-bottom: 60rpx;
}

.header {
  text-align: center;
  margin-bottom: 30rpx;
}

.title {
  display: block;
  font-size: 48rpx;
  font-weight: bold;
  color: #ffffff;
  text-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.subtitle {
  display: block;
  font-size: 26rpx;
  color: rgba(255,255,255,0.8);
  margin-top: 8rpx;
}

.card {
  background: #ffffff;
  border-radius: 20rpx;
  padding: 30rpx;
  margin-bottom: 24rpx;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
}

.action-card {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
}

.section-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
  margin-bottom: 20rpx;
  padding-left: 16rpx;
  border-left: 6rpx solid #667eea;
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: #fff;
  border: none;
  border-radius: 16rpx;
  padding: 24rpx 40rpx;
  font-size: 32rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12rpx;
}

.btn-primary::after {
  border: none;
}

.btn-icon {
  font-size: 40rpx;
  font-weight: bold;
}

.thumb-list {
  display: flex;
  flex-wrap: wrap;
  margin-top: 24rpx;
  gap: 16rpx;
}

.thumb-wrap {
  position: relative;
}

.thumb {
  width: 180rpx;
  height: 180rpx;
  border-radius: 12rpx;
  background-color: #f0f0f0;
}

.thumb-index {
  position: absolute;
  top: 8rpx;
  left: 8rpx;
  background: rgba(0,0,0,0.6);
  color: #fff;
  font-size: 22rpx;
  width: 36rpx;
  height: 36rpx;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.thumb-check {
  position: absolute;
  top: 8rpx;
  right: 8rpx;
  width: 40rpx;
  height: 40rpx;
  border-radius: 50%;
  border: 2rpx solid rgba(255,255,255,0.8);
  background: rgba(0,0,0,0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
  font-size: 24rpx;
  font-weight: bold;
}

.thumb-check.active {
  background: #007aff;
  border-color: #007aff;
}

.thumb-check-placeholder {
  position: absolute;
  top: 8rpx;
  right: 8rpx;
  width: 40rpx;
  height: 40rpx;
  border-radius: 50%;
  border: 2rpx solid rgba(255,255,255,0.5);
  background: rgba(0,0,0,0.1);
}

.delete-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 20rpx;
  padding-top: 20rpx;
  border-top: 1rpx solid #eee;
}

.delete-info {
  font-size: 28rpx;
  color: #666;
}

.delete-actions {
  display: flex;
  gap: 16rpx;
}

.btn-cancel {
  background: #f0f0f0;
  color: #666;
  border: none;
  border-radius: 12rpx;
  padding: 12rpx 24rpx;
  font-size: 26rpx;
}

.btn-cancel::after {
  border: none;
}

.btn-delete {
  background: #ff3b30;
  color: #fff;
  border: none;
  border-radius: 12rpx;
  padding: 12rpx 24rpx;
  font-size: 26rpx;
}

.btn-delete::after {
  border: none;
}

.empty {
  text-align: center;
  color: #999;
  margin-top: 30rpx;
  font-size: 28rpx;
  padding: 40rpx 0;
}

.preset-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 20rpx;
}

.picker-box {
  flex: 1;
  background: #f8f9fa;
  border-radius: 12rpx;
  padding: 20rpx 24rpx;
  display: flex;
  align-items: center;
}

.picker-label {
  font-size: 28rpx;
  color: #666;
}

.picker-value {
  flex: 1;
  font-size: 30rpx;
  color: #333;
  margin: 0 12rpx;
}

.picker-arrow {
  font-size: 24rpx;
  color: #999;
}

.btn-mini {
  background: #f0f0f0;
  color: #333;
  border: none;
  border-radius: 12rpx;
  padding: 16rpx 24rpx;
  font-size: 26rpx;
  white-space: nowrap;
}

.btn-mini::after {
  border: none;
}

.form-item {
  margin-bottom: 24rpx;
}

.form-item:last-child {
  margin-bottom: 0;
}

.label-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12rpx;
}

.label {
  display: block;
  font-size: 28rpx;
  color: #555;
  margin-bottom: 12rpx;
  font-weight: 500;
}

.label-row .label {
  margin-bottom: 0;
}

.history-toggle {
  display: flex;
  align-items: center;
  gap: 8rpx;
  padding: 8rpx 16rpx;
  background: #f0f0f0;
  border-radius: 12rpx;
}

.history-toggle-text {
  font-size: 24rpx;
  color: #007aff;
}

.history-toggle-icon {
  font-size: 20rpx;
  color: #999;
  transition: transform 0.3s;
}

.history-toggle-icon.open {
  transform: rotate(180deg);
}

.input {
  background: #f8f9fa;
  border-radius: 12rpx;
  padding: 20rpx 24rpx;
  font-size: 30rpx;
  color: #333;
  border: 2rpx solid transparent;
  transition: border-color 0.3s;
}

.input:focus {
  border-color: #667eea;
}

.history-dropdown {
  margin-top: 12rpx;
  background: #f8f9fa;
  border-radius: 12rpx;
  overflow: hidden;
}

.history-item {
  display: flex;
  align-items: center;
  padding: 20rpx 24rpx;
  border-bottom: 1rpx solid #eee;
  transition: background 0.2s;
}

.history-item:last-child {
  border-bottom: none;
}

.history-item:active {
  background: #e6f2ff;
}

.history-rank {
  width: 36rpx;
  height: 36rpx;
  background: #007aff;
  color: #fff;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 22rpx;
  margin-right: 16rpx;
}

.history-text {
  flex: 1;
  font-size: 28rpx;
  color: #333;
}

.history-freq {
  font-size: 24rpx;
  color: #999;
}

.preview-box {
  margin-top: 16rpx;
  padding: 20rpx;
  background: #f8f9fa;
  border-radius: 12rpx;
  display: flex;
  align-items: center;
  gap: 16rpx;
}

.preview-label {
  font-size: 26rpx;
  color: #999;
  white-space: nowrap;
}

.preview-text {
  color: #333;
  font-weight: 500;
}

.font-preview-list {
  display: flex;
  flex-wrap: wrap;
  gap: 16rpx;
  margin-top: 16rpx;
}

.font-preview-item {
  padding: 16rpx 24rpx;
  background: #f8f9fa;
  border-radius: 12rpx;
  font-size: 28rpx;
  color: #666;
  border: 2rpx solid transparent;
  transition: all 0.3s;
}

.font-preview-item.active {
  background: #e6f2ff;
  color: #007aff;
  border-color: #007aff;
}

.switch-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f8f9fa;
  border-radius: 12rpx;
  padding: 20rpx 24rpx;
}

.switch-info {
  flex: 1;
}

.switch-info .label {
  margin-bottom: 4rpx;
}

.switch-desc {
  font-size: 24rpx;
  color: #999;
}

.btn-process {
  width: 100%;
  background: #ffffff;
  color: #f5576c;
  border: none;
  border-radius: 16rpx;
  padding: 32rpx 0;
  font-size: 36rpx;
  font-weight: 600;
}

.btn-process::after {
  border: none;
}

.btn-process.disabled {
  opacity: 0.6;
}

.platform-tip {
  text-align: center;
  margin-top: 20rpx;
}

.platform-tip text {
  font-size: 26rpx;
  color: rgba(255,255,255,0.9);
}

.preview-actions {
  display: flex;
  flex-direction: row;
  gap: 16rpx;
  margin-top: 24rpx;
}

.preview-actions button {
  flex: 1;
  border: none;
  border-radius: 12rpx;
  padding: 20rpx 0;
  font-size: 28rpx;
  font-weight: 500;
  text-align: center;
}

.preview-actions button::after {
  border: none;
}

.btn-save {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: #fff;
}

.btn-refresh {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  color: #fff;
}

.btn-clear {
  background: #f0f0f0;
  color: #666;
}
</style>
