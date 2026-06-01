# 图片水印工具 - 开发报告

## 项目概述

**项目名称**：Open Study For Uni  
**项目类型**：UniApp 跨平台应用（Android App + H5 网页）  
**开发时间**：2025年5月 - 2025年6月  
**主要功能**：为图片批量添加时间地点水印

---

## 一、开发环境与技术栈

### 1.1 技术栈
- **前端框架**：UniApp（基于 Vue 3）
- **开发工具**：HBuilderX
- **目标平台**：Android App、iOS App、H5 网页
- **版本控制**：Git + GitHub

### 1.2 核心 API
- `uni.chooseImage` - 图片选择
- `uni.getImageInfo` - 获取图片信息
- `uni.createCanvasContext` - 创建 Canvas 上下文
- `uni.canvasToTempFilePath` - Canvas 导出图片
- `uni.saveImageToPhotosAlbum` - 保存到相册
- `uni.getSavedFileInfo` / `uni.removeSavedFile` - 文件管理

---

## 二、开发过程中遇到的问题及解决方案

### 问题 1：App 端白屏报错 "WeexPlus is not a constructor"

**问题描述**：  
应用安装到手机后打开直接白屏，控制台报错 `WeexPlus is not a constructor`。

**问题原因**：  
AI 生成的代码中使用了网页端专用的 DOM API（如 `document.createElement`、`window` 等），在 App 端的 V8/Weex 引擎中不存在这些对象，导致 JS 引擎崩溃。

**解决方案**：  
1. 删除所有 `document.createElement` 相关代码
2. 使用 UniApp 标准 Canvas API：
   - 在 `<template>` 中定义 `<canvas canvas-id="watermarkCanvas">`
   - 通过 `uni.createCanvasContext('watermarkCanvas')` 获取上下文
   - 使用 `ctx.drawImage`、`ctx.setFontSize`、`ctx.fillText` 等标准方法
3. 确保代码中没有使用任何 DOM API

**关键代码**：
```vue
<template>
  <canvas canvas-id="watermarkCanvas" :style="canvasStyle"></canvas>
</template>

<script>
const ctx = uni.createCanvasContext('watermarkCanvas')
ctx.drawImage(src, 0, 0, w, h)
ctx.setFontSize(fontSize)
ctx.fillText(text, x, y)
ctx.draw(false, () => {
  uni.canvasToTempFilePath({
    canvasId: 'watermarkCanvas',
    success: (res) => {
      // res.tempFilePath 为生成的临时图片路径
    }
  })
})
</script>
```

---

### 问题 2：保存的图片是纯黑色或纯白色

**问题描述**：  
处理后的图片保存到相册后，显示为纯黑色或纯白色。

**问题原因**：  
Canvas 组件的宽高被固定为 1px，导致绘制区域过小，无法正常渲染图片内容。

**解决方案**：  
动态绑定 Canvas 尺寸，根据原图宽高设置：
```javascript
const canvasWidth = ref(1)
const canvasHeight = ref(1)

const canvasStyle = computed(() => {
  return `position: fixed; left: -9999px; width: ${canvasWidth.value}px; height: ${canvasHeight.value}px;`
})

// 处理图片时动态设置尺寸
function processImage(src) {
  uni.getImageInfo({
    src,
    success: (imgInfo) => {
      canvasWidth.value = imgInfo.width
      canvasHeight.value = imgInfo.height
      // ... 绘制逻辑
    }
  })
}
```

---

### 问题 3：第二张及后续图片出现切图/裁剪

**问题描述**：  
处理多张图片时，第一张正常，但第二张及之后的图片出现内容被裁剪或显示不全。

**问题原因**：  
Canvas 尺寸未在处理每张图片时重新设置，导致后续图片使用了前一张图片的尺寸。

**解决方案**：  
在 `processImage` 函数中，每次处理新图片时都重新设置 Canvas 宽高：
```javascript
function processImage(src, index, callback) {
  uni.getImageInfo({
    src,
    success: (imgInfo) => {
      let w = imgInfo.width
      let h = imgInfo.height
      
      // 限制最大尺寸以提高性能
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

      // 每次处理都重新设置 Canvas 尺寸
      canvasWidth.value = w
      canvasHeight.value = h
      
      const ctx = uni.createCanvasContext('watermarkCanvas')
      ctx.drawImage(src, 0, 0, w, h)
      // ... 绘制水印
    }
  })
}
```

---

### 问题 4：字体修改后水印字体未生效

**问题描述**：  
在设置中修改了字体，但生成的水印文字字体没有变化。

**问题原因**：  
绘制水印时只设置了字体大小，没有设置字体名称。

**解决方案**：  
在 `drawStrokeText` 函数中添加字体设置：
```javascript
function drawStrokeText(ctx, text, x, y, fontSize, fontName) {
  ctx.font = `${fontSize}px ${fontName}`  // 设置字体
  ctx.setFontSize(fontSize)
  
  // 绘制描边
  ctx.setFillStyle('#000000')
  ctx.setStrokeStyle('#000000')
  ctx.setLineWidth(fontSize * 0.15)
  ctx.strokeText(text, x, y)
  ctx.fillText(text, x, y)
  
  // 绘制填充
  ctx.setFillStyle('#ffffff')
  ctx.fillText(text, x, y)
}
```

---

### 问题 5：UI 库加载失败，部分核心库未加载

**问题描述**：  
应用启动时提示 "UI 库加载失败" 或 "部分核心库未加载"。

**问题原因**：  
1. `manifest.json` 缺少必要的权限声明
2. 使用了不兼容的 API 参数（如 `geocode: true`）

**解决方案**：  
1. 在 `manifest.json` 中添加定位权限：
```json
"permissions": {
  "Geolocation": {
    "description": "获取位置信息"
  }
}
```

2. 简化定位逻辑，去掉不兼容参数：
```javascript
// 错误代码
uni.getLocation({
  type: 'gcj02',
  geocode: true,  // 此参数在 App 端不支持
  success: () => {}
})

// 正确代码
uni.getLocation({
  type: 'gcj02',
  success: () => {}
})
```

---

### 问题 6：图片越存越多，相册堆积大量重复图片

**问题描述**：  
每次点击"批量处理"生成预览后，手机相册里就会出现新图片，导致重复图片堆积。

**问题原因**：  
在 `processImage` 函数中，生成预览后错误地调用了 `saveFile`，而 `saveFile` 函数在 App 端直接调用了 `uni.saveImageToPhotosAlbum`，导致预览时就保存到了相册。

**解决方案**：  
重构保存逻辑，实现"先缓存预览，后手动存相册"：

**修改前（错误逻辑）**：
```javascript
// processImage 函数
uni.canvasToTempFilePath({
  success: (res) => {
    saveFile(res.tempFilePath, index, () => {
      // 预览阶段就保存到相册了！
    })
  }
})
```

**修改后（正确逻辑）**：
```javascript
// processImage 函数 - 只生成预览，不保存
uni.canvasToTempFilePath({
  success: (res) => {
    // 只返回临时文件路径，不保存到相册
    if (callback) callback(res.tempFilePath)
  }
})

// saveAllImages 函数 - 只在点击保存时写入相册
function saveAllImages() {
  resultImages.value.forEach((path, index) => {
    uni.saveImageToPhotosAlbum({
      filePath: path,
      success: () => {}
    })
  })
}
```

---

### 问题 7：刷新后旧预览图片未删除，占用存储空间

**问题描述**：  
点击"刷新"重新生成预览后，之前的预览图片文件仍然残留在手机存储中。

**问题原因**：  
刷新时只清空了数组，没有删除对应的临时文件。

**解决方案**：  
1. 刷新前删除旧的临时文件：
```javascript
function refreshAllImages() {
  // 删除之前的预览缓存文件
  previewImages.value.forEach(path => {
    deleteTempFile(path)
  })
  previewImages.value = []
  // ... 重新生成
}
```

2. 使用 `plus.io` 删除临时文件：
```javascript
function deleteTempFile(filePath) {
  if (!filePath) return
  // #ifdef APP-PLUS
  plus.io.resolveLocalFileSystemURL(filePath, (entry) => {
    entry.remove(() => {
      console.log('临时文件已删除:', filePath)
    })
  })
  // #endif
}
```

---

### 问题 8：保存后再次处理，会保存上一次的旧图片

**问题描述**：  
保存完成后，再次选择新图片处理并保存，结果相册里同时出现了新图片和上一次的老图片。

**问题原因**：  
状态管理问题，`resultImages` 数组在每次处理前没有被清空，导致保存时遍历了旧数据。

**解决方案**：  
1. 处理前清空所有结果数组：
```javascript
function batchWatermark() {
  // 清空之前的结果数组，避免旧数据干扰
  previewImages.value = []
  resultImages.value = []
  processing.value = true
  // ... 开始处理
}
```

2. 保存时只遍历最新的 `resultImages`：
```javascript
function saveAllImages() {
  // 只保存当前最新的预览图片
  resultImages.value = [...previewImages.value]
  
  resultImages.value.forEach((path, index) => {
    saveFile(path, index, () => {
      // ...
    })
  })
}
```

3. 保存完成后彻底释放资源：
```javascript
function releaseResources() {
  previewImages.value = []
  resultImages.value = []
  images.value = []
  selectedIndexes.value = []
  isMultiSelectMode.value = false
}
```

---

### 问题 9：启动页图片被拉伸变形

**问题描述**：  
App 启动时的加载页图片被拉伸或缩放，显示效果不佳。

**问题原因**：  
`manifest.json` 中 `splashscreen` 的 `androidStyle` 设置为 `default`，导致图片被拉伸填充。

**解决方案**：  
将 `androidStyle` 改为 `common`，实现图片从中间裁剪显示：
```json
"splashscreen": {
  "androidStyle": "common",
  "android": {
    "hdpi": "static/splash.png",
    "xhdpi": "static/splash.png",
    "xxhdpi": "static/splash.png"
  }
}
```

---

### 问题 10：地点输入体验不佳

**问题描述**：  
用户每次都需要手动输入地点，没有历史记录功能。

**解决方案**：  
1. 使用 `uni.getStorage` / `uni.setStorage` 保存历史记录
2. 按使用频率排序，常用地点排在前面
3. 添加折叠选项框展示历史记录：
```javascript
const historyLocations = ref([])
const locationFreq = ref({})

function addToHistory(loc) {
  if (!loc) return
  locationFreq.value[loc] = (locationFreq.value[loc] || 0) + 1
  
  if (!historyLocations.value.includes(loc)) {
    historyLocations.value.push(loc)
  }
  
  // 按使用频率排序
  historyLocations.value.sort((a, b) => {
    return (locationFreq.value[b] || 0) - (locationFreq.value[a] || 0)
  })
  
  uni.setStorageSync('history', historyLocations.value)
}
```

---

### 问题 11：移动端双击事件不生效

**问题描述**：  
在 App 端双击预览图片重新处理的功能无法触发。

**问题原因**：  
移动端浏览器和 App 环境不支持原生的 `dblclick` 事件。

**解决方案**：  
使用自定义双击检测逻辑：
```javascript
const lastClickTime = ref(0)
const lastClickIndex = ref(-1)

function handleImageClick(index) {
  const currentTime = Date.now()
  const timeDiff = currentTime - lastClickTime.value

  if (timeDiff < 300 && lastClickIndex.value === index) {
    // 300ms 内双击同一图片，触发重新处理
    reprocessImage(index)
  } else {
    // 单击，预览图片
    previewImage(index)
  }

  lastClickTime.value = currentTime
  lastClickIndex.value = index
}
```

---

### 问题 12：图片处理速度慢

**问题描述**：  
处理大尺寸图片时速度很慢，用户体验不佳。

**解决方案**：  
1. 限制图片最大尺寸：
```javascript
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
```

2. 降低导出质量（在可接受范围内）：
```javascript
uni.canvasToTempFilePath({
  quality: 0.9,  // 适当降低质量以提高速度
  fileType: 'jpg',
  // ...
})
```

---

## 三、最佳实践总结

### 3.1 UniApp App 端开发注意事项

1. **禁止使用 DOM API**：`document`、`window`、`localStorage` 等在 App 端不可用
2. **使用条件编译**：`// #ifdef APP-PLUS` 区分 App 端和 H5 端代码
3. **Canvas 使用**：必须在 template 中定义，通过 `uni.createCanvasContext` 获取
4. **文件管理**：使用 `uni.getSavedFileList` 和 `uni.removeSavedFile` 管理缓存

### 3.2 状态管理规范

1. **处理前清空**：每次开始新处理前，清空所有相关数组
2. **保存后释放**：保存完成后立即释放资源，避免内存泄漏
3. **临时文件管理**：及时删除不再需要的临时文件

### 3.3 用户体验优化

1. **先预览后保存**：避免用户不满意的结果直接保存到相册
2. **进度提示**：使用 `uni.showLoading` 显示处理进度
3. **错误处理**：每个异步操作都要有 `fail` 回调
4. **操作反馈**：使用 `uni.showToast` 给用户即时反馈

---

## 四、项目文件结构

```
S_ad-time/
├── pages/
│   └── index/
│       └── index.vue          # 主页面（核心功能）
├── static/
│   └── logo.png               # 应用图标
├── App.vue                    # 应用入口
├── main.js                    # 主入口文件
├── manifest.json              # 应用配置（权限、启动页等）
├── pages.json                 # 页面配置
├── uni.scss                   # 全局样式
├── README.md                  # 项目说明文档
└── DEVELOPMENT_REPORT.md      # 本开发报告
```

---

## 五、Git 提交记录

| 提交时间 | 提交信息 | 说明 |
|---------|---------|------|
| 2025-05-30 | init: 项目初始化 | 初始版本 |
| 2025-05-31 | fix: 修复白屏问题 | 移除 DOM API，使用标准 Canvas |
| 2025-06-01 | feat: 添加预览保存功能 | 实现先预览后保存 |
| 2025-06-01 | fix: 修复临时文件未正确删除 | 优化缓存管理 |
| 2025-06-01 | fix: 修复状态管理问题 | 确保每次处理前清空结果数组 |
| 2025-06-01 | fix: 重构保存逻辑 | 预览时不保存到相册 |

---

## 六、未来优化方向

1. **性能优化**：使用 Web Worker 处理图片，避免阻塞主线程
2. **功能扩展**：支持自定义水印位置、透明度、旋转角度
3. **批量操作**：支持拖拽排序、批量删除
4. **云端同步**：支持将配置和历史记录同步到云端
5. **更多平台**：支持微信小程序、支付宝小程序等

---

**报告编写时间**：2025年6月1日  
**编写人**：AI 开发助手  
**项目地址**：https://github.com/Siqihub/Open-study-for-uni
