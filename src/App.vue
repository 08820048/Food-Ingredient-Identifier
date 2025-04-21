<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { UploadFilled, Delete, Grid, List, Loading, Picture, Search, InfoFilled } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import axios from 'axios'
import { marked } from 'marked'
import { driver } from 'driver.js'
import 'driver.js/dist/driver.css'

const imageFile = ref(null)
const imageUrl = ref('')
const loading = ref(false)
const result = ref('')
const layout = ref('vertical')
const analysisProgress = ref(0)
const progressInterval = ref(null)
const analysisSteps = ref([
  { text: '初始化分析引擎...', done: false },
  { text: '加载图像识别模型...', done: false },
  { text: '提取图像特征...', done: false },
  { text: '分析食品成分...', done: false },
  { text: '生成分析报告...', done: false }
])

// 初始化引导
const guide = driver({
  animate: true,
  opacity: 0.75,
  padding: 10,
  showButtons: ['close', 'next'],
  showProgress: true,
  steps: [
    {
      element: '.upload-area',
      popover: {
        title: '上传图片',
        description: '点击这里或拖放图片来上传需要分析的食品图片',
        position: 'bottom'
      }
    },
    {
      element: '.analyze-button',
      popover: {
        title: '开始分析',
        description: '上传图片后点击这里开始AI分析',
        position: 'top'
      }
    },
    {
      element: '.result-panel',
      popover: {
        title: '分析结果',
        description: '这里将显示AI分析的结果，包括食品成分和营养价值等信息',
        position: 'left'
      }
    },
    {
      element: '.layout-switch',
      popover: {
        title: '布局切换',
        description: '可以在这里切换竖版或横版布局',
        position: 'right'
      }
    }
  ]
})

// 开始引导
const startGuide = () => {
  guide.drive()
}

// 模拟分析进度
const simulateAnalysisProgress = () => {
  // 重置状态
  analysisSteps.value.forEach(step => step.done = false)
  analysisProgress.value = 0
  
  // 清除可能存在的旧定时器
  if (progressInterval.value) {
    clearInterval(progressInterval.value)
  }
  
  const stepTime = 800 // 每个步骤的基础时间
  let currentStep = 0
  
  const updateProgress = () => {
    if (currentStep >= analysisSteps.value.length) {
      clearInterval(progressInterval.value)
      progressInterval.value = null
      return
    }

    const targetProgress = (currentStep + 1) * (100 / analysisSteps.value.length)
    const smoothProgress = () => {
      const diff = targetProgress - analysisProgress.value
      const increment = diff * 0.1
      
      if (Math.abs(diff) < 0.1) {
        analysisProgress.value = targetProgress
        analysisSteps.value[currentStep].done = true
        currentStep++
        setTimeout(updateProgress, 100)
      } else {
        analysisProgress.value += increment
        requestAnimationFrame(smoothProgress)
      }
    }
    
    smoothProgress()
  }
  
  updateProgress()
}

// 清理函数
const cleanupAnalysis = () => {
  if (progressInterval.value) {
    clearInterval(progressInterval.value)
    progressInterval.value = null
  }
  analysisSteps.value.forEach(step => step.done = false)
  analysisProgress.value = 0
}

// 处理文件变化
const handleFileChange = (file) => {
  const isImage = file.raw.type.startsWith('image/')
  if (!isImage) {
    ElMessage.error('请上传图片文件！')
    return
  }

  imageFile.value = file.raw
  imageUrl.value = URL.createObjectURL(file.raw)
  result.value = ''
}

// 清除图片
const clearImage = () => {
  imageFile.value = null
  imageUrl.value = ''
  result.value = ''
}

// 分析图片
const analyzeImage = async () => {
  if (!imageFile.value) {
    ElMessage.warning('请先上传图片')
    return
  }

  loading.value = true
  result.value = ''
  simulateAnalysisProgress()
  
  try {
    const reader = new FileReader()
    reader.readAsDataURL(imageFile.value)
    reader.onload = async () => {
      try {
        const response = await axios.post('/api/analyze', {
          image: reader.result
        })

        result.value = response.data.result
        ElMessage.success('分析完成')
      } catch (error) {
        console.error('API 错误:', error)
        ElMessage.error(error.response?.data?.error || '分析失败，请稍后重试')
        result.value = ''
      } finally {
        cleanupAnalysis()
        loading.value = false
      }
    }

    reader.onerror = () => {
      ElMessage.error('图片读取失败')
      cleanupAnalysis()
      loading.value = false
    }
  } catch (error) {
    console.error('处理错误:', error)
    ElMessage.error('图片处理失败')
    cleanupAnalysis()
    loading.value = false
  }
}

// 渲染Markdown内容
const renderMarkdown = (text) => {
  return marked(text, {
    breaks: true,
    gfm: true
  })
}

// 切换布局
const toggleLayout = () => {
  layout.value = layout.value === 'vertical' ? 'horizontal' : 'vertical'
}
</script>

<template>
  <div class="app-container">
    <header class="app-header">
      <div class="header-content">
        <div class="logo">
          <el-icon class="logo-icon"><Picture /></el-icon>
          <h1>Food Ingredient Identifier</h1>
        </div>
        <p class="subtitle">智能食品成分分析系统</p>
        <el-button
          class="guide-button"
          type="info"
          circle
          @click="startGuide"
          title="使用引导"
        >
          <el-icon><InfoFilled /></el-icon>
        </el-button>
      </div>
    </header>

    <main class="app-main">
      <div class="main-content" :class="layout">
        <!-- 左侧操作面板 -->
        <div class="operation-panel">
          <el-card class="upload-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <h2>上传图片</h2>
                <el-button-group class="layout-switch">
                  <el-button
                    :type="layout === 'vertical' ? 'primary' : 'default'"
                    @click="toggleLayout"
                    :icon="Grid"
                    title="竖版布局"
                  />
                  <el-button
                    :type="layout === 'horizontal' ? 'primary' : 'default'"
                    @click="toggleLayout"
                    :icon="List"
                    title="横版布局"
                  />
                </el-button-group>
              </div>
            </template>

            <el-upload
              v-if="!imageUrl"
              class="upload-area"
              drag
              action=""
              :auto-upload="false"
              :show-file-list="false"
              accept="image/*"
              :on-change="handleFileChange"
              :disabled="loading"
            >
              <template #trigger>
                <div class="upload-content">
                  <el-icon class="el-icon--upload"><upload-filled /></el-icon>
                  <div class="el-upload__text">
                    拖放图片到此处或 <em>点击上传</em>
                  </div>
                  <div class="upload-requirements">
                    <p class="requirement-title">图片要求：</p>
                    <ul class="requirement-list">
                      <li>支持 JPG、PNG、GIF 等常见格式</li>
                      <li>建议图片分辨率不低于 800x600</li>
                      <li>图片大小不超过 5MB</li>
                      <li>尽量保证食品标签清晰可见</li>
                    </ul>
                  </div>
                </div>
              </template>
            </el-upload>

            <div v-if="imageUrl" class="preview-container">
              <div class="preview-header">
                <h3>已上传图片</h3>
                <el-button
                  type="danger"
                  circle
                  class="clear-button"
                  @click="clearImage"
                  :disabled="loading"
                  title="删除图片"
                >
                  <el-icon><delete /></el-icon>
                </el-button>
              </div>
              <div class="preview-wrapper">
                <el-image
                  :src="imageUrl"
                  fit="contain"
                  class="preview-image"
                  :preview-src-list="[imageUrl]"
                  :initial-index="0"
                  preview-teleported
                />
              </div>
              <div class="preview-footer">
                <p class="preview-tip">点击图片可查看大图</p>
              </div>
            </div>

            <el-button
              type="primary"
              :loading="loading"
              class="analyze-button"
              @click="analyzeImage"
              :disabled="!imageFile || loading"
            >
              <el-icon v-if="loading" class="is-loading"><loading /></el-icon>
              <el-icon v-else class="el-icon--right"><Search /></el-icon>
              {{ loading ? '分析中...' : '开始分析' }}
            </el-button>
          </el-card>
        </div>

        <!-- 右侧结果面板 -->
        <div class="result-panel">
          <el-card class="result-card" shadow="hover">
            <template #header>
              <div class="card-header">
                <h2>分析结果</h2>
                <el-tag v-if="result" type="success">分析完成</el-tag>
                <el-tag v-else-if="loading" type="warning">分析中</el-tag>
                <el-tag v-else type="info">等待分析</el-tag>
              </div>
            </template>

            <div v-if="loading" class="loading-content">
              <el-icon class="loading-icon"><loading /></el-icon>
              <p class="loading-text">AI正在分析图片内容，请稍候...</p>
              <p class="loading-subtext">这可能需要几秒钟时间</p>
            </div>
            <div v-else-if="result" class="result-content markdown-body" v-html="renderMarkdown(result)"></div>
            <div v-else class="placeholder-content">
              <el-icon class="placeholder-icon"><Search /></el-icon>
              <p class="placeholder-text">上传图片并点击"开始分析"按钮</p>
              <p class="placeholder-subtext">AI将为您详细解读图片内容</p>
            </div>
          </el-card>
        </div>
      </div>
    </main>

    <footer class="app-footer">
      <p>© 2024 Food Ingredient Identifier. All rights reserved.</p>
    </footer>

    <!-- 分析进度模态框 -->
    <el-dialog
      v-model="loading"
      :show-close="false"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
      width="400px"
      class="analysis-dialog"
      align-center
    >
      <div class="analysis-content">
        <div class="analysis-header">
          <div class="analysis-title">AI分析进行中</div>
          <div class="analysis-subtitle">请稍候，正在处理您的图片...</div>
        </div>
        
        <div class="progress-container">
          <div class="progress-text">{{ Math.round(analysisProgress) }}%</div>
          <el-progress
            :percentage="analysisProgress"
            :stroke-width="8"
            :show-text="false"
            class="analysis-progress"
          />
        </div>

        <div class="analysis-steps">
          <div
            v-for="(step, index) in analysisSteps"
            :key="index"
            class="step-item"
            :class="{ 'step-done': step.done }"
          >
            <div class="step-indicator">
              <div class="step-dot"></div>
              <div class="step-line" v-if="index < analysisSteps.length - 1"></div>
            </div>
            <div class="step-text">{{ step.text }}</div>
          </div>
        </div>

        <div class="analysis-decoration">
          <div class="tech-grid"></div>
          <div class="spectrum-container">
            <div class="spectrum-bar" v-for="n in 12" :key="n"></div>
          </div>
        </div>
      </div>
    </el-dialog>
  </div>
</template>

<style>
.app-container {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(135deg, var(--el-bg-color) 0%, var(--el-bg-color-page) 100%);
  color: var(--el-text-color-primary);
  transition: all 0.3s ease;
}

.app-header {
  background: var(--el-bg-color);
  padding: 20px 0;
  box-shadow: 0 2px 12px 0 var(--el-box-shadow);
  position: relative;
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 0 20px;
  text-align: center;
  position: relative;
}

.logo-icon {
  font-size: 32px;
  color: var(--el-color-primary);
}

h1 {
  margin: 0;
  font-size: 28px;
  color: var(--el-text-color-primary);
}

.subtitle {
  color: var(--el-text-color-secondary);
  font-size: 16px;
  margin: 0;
}

.app-main {
  flex: 1;
  padding: 40px 20px;
}

.main-content {
  max-width: 1200px;
  margin: 0 auto;
  display: flex;
  gap: 20px;
}

.main-content.vertical {
  flex-direction: column;
}

.main-content.horizontal {
  flex-direction: row;
}

.operation-panel {
  flex: 1;
  min-width: 0;
}

.result-panel {
  flex: 1;
  min-width: 0;
}

.upload-card, .result-card {
  height: 100%;
  border-radius: 12px;
  transition: all 0.3s ease;
  background: var(--el-bg-color);
  border: 1px solid var(--el-border-color-light);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid var(--el-border-color-light);
}

h2 {
  margin: 0;
  font-size: 20px;
  color: var(--el-text-color-primary);
}

.upload-area {
  width: 100%;
  margin-bottom: 20px;
  background: var(--upload-area-bg, rgba(0, 0, 0, 0.02));
  border: 2px dashed var(--el-border-color);
}

.upload-content {
  padding: 40px 0;
  text-align: center;
  color: var(--el-text-color-regular);
}

.el-icon--upload {
  font-size: 48px;
  color: var(--el-color-primary);
  margin-bottom: 16px;
}

.el-upload__text {
  font-size: 16px;
  margin: 16px 0;
}

.upload-requirements {
  margin-top: 16px;
  padding: 12px 16px;
  background: var(--requirements-bg, rgba(0, 0, 0, 0.03));
  border-radius: 8px;
  text-align: left;
}

.requirement-title {
  font-size: 14px;
  color: var(--el-text-color-regular);
  margin-bottom: 8px;
  font-weight: 500;
}

.requirement-list {
  list-style: none;
  padding: 0;
  margin: 0;
  font-size: 13px;
  color: var(--el-text-color-secondary);
}

.requirement-list li {
  margin: 4px 0;
  padding-left: 20px;
  position: relative;
  line-height: 1.4;
}

.requirement-list li::before {
  content: '•';
  position: absolute;
  left: 8px;
  color: cornflowerblue;
}

.preview-container {
  background: #fff;
  border-radius: 12px;
  padding: 20px;
  margin: 20px 0;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.preview-container:hover {
  box-shadow: 0 4px 16px 0 rgba(0, 0, 0, 0.15);
  transform: translateY(-2px);
}

.preview-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid #ebeef5;
}

.preview-header h3 {
  margin: 0;
  font-size: 18px;
  color: #303133;
  font-weight: 600;
}

.preview-wrapper {
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  background: #f5f7fa;
  padding: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 300px;
}

.preview-image {
  max-width: 100%;
  max-height: 400px;
  object-fit: contain;
  border-radius: 4px;
  transition: all 0.3s ease;
  cursor: zoom-in;
}

.preview-image:hover {
  transform: scale(1.02);
}

.preview-footer {
  margin-top: 12px;
  text-align: center;
}

.preview-tip {
  color: #909399;
  font-size: 14px;
  margin: 0;
}

.clear-button {
  opacity: 0.8;
  transition: all 0.3s ease;
}

.clear-button:hover {
  opacity: 1;
  transform: scale(1.1);
  background-color: #f56c6c;
  color: white;
}

.analyze-button {
  width: 100%;
  height: 48px;
  font-size: 16px;
  border-radius: 8px;
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.analyze-button:not(:disabled):hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(64, 158, 255, 0.3);
}

.analyze-button:disabled {
  cursor: not-allowed;
  opacity: 0.7;
  background-color: var(--el-button-disabled-bg-color);
  border-color: var(--el-button-disabled-border-color);
}

.analyze-button .el-icon {
  margin: 0;
  font-size: 18px;
}

.loading-content, .placeholder-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 40px 20px;
  text-align: center;
  color: #909399;
  flex: 1;
  min-height: 300px;
}

.loading-icon {
  font-size: 48px;
  margin-bottom: 16px;
  color: #409EFF;
  animation: rotating 2s linear infinite;
}

.loading-text {
  font-size: 16px;
  color: #606266;
  margin-bottom: 8px;
}

.loading-subtext {
  font-size: 14px;
  color: #909399;
}

.placeholder-icon {
  font-size: 48px;
  margin-bottom: 16px;
  color: #dcdfe6;
}

.placeholder-text {
  font-size: 16px;
  margin-bottom: 8px;
  color: #606266;
}

.placeholder-subtext {
  font-size: 14px;
  color: #909399;
}

.result-content {
  white-space: pre-line;
  line-height: 1.6;
  color: #606266;
  padding: 20px;
  flex: 1;
  min-height: 300px;
  background-color: #ffffff;
  background-image: 
    linear-gradient(#e5e5e5 1px, transparent 1px),
    linear-gradient(90deg, #e5e5e5 1px, transparent 1px);
  background-size: 20px 20px;
  border-radius: 8px;
}

.result-content.markdown-body {
  background-color: #ffffff;
  background-image: 
    linear-gradient(rgba(229, 229, 229, 0.3) 1px, transparent 1px),
    linear-gradient(90deg, rgba(229, 229, 229, 0.3) 1px, transparent 1px);
  background-size: 20px 20px;
  box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.05);
}

.app-footer {
  background: rgba(255, 255, 255, 0.9);
  padding: 20px 0;
  text-align: center;
  color: #909399;
  font-size: 14px;
}

@keyframes rotating {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

:deep(.el-upload-dragger) {
  width: 100%;
  border: 2px dashed #dcdfe6;
  border-radius: 8px;
  transition: all 0.3s ease;
}

:deep(.el-upload-dragger:hover) {
  border-color: #409EFF;
  background-color: rgba(64, 158, 255, 0.05);
}

:deep(.el-upload-dragger.is-dragover) {
  border-color: #409EFF;
  background-color: rgba(64, 158, 255, 0.1);
}

:deep(.el-upload-dragger.is-disabled) {
  opacity: 0.6;
  cursor: not-allowed;
}

:deep(.el-button.is-disabled) {
  opacity: 0.6;
  cursor: not-allowed;
}

@media (max-width: 768px) {
  .main-content {
    flex-direction: column;
  }
  
  .app-header {
    padding: 15px 0;
  }
  
  h1 {
    font-size: 24px;
  }
  
  .subtitle {
    font-size: 14px;
  }
  
  .preview-image {
    max-height: 300px;
  }
}

.markdown-body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
  font-size: 16px;
  line-height: 1.5;
  color: #24292e;
  padding: 20px;
}

.markdown-body :deep(h1),
.markdown-body :deep(h2),
.markdown-body :deep(h3),
.markdown-body :deep(h4),
.markdown-body :deep(h5),
.markdown-body :deep(h6) {
  margin-top: 16px;
  margin-bottom: 12px;
  font-weight: 600;
  line-height: 1.25;
}

.markdown-body :deep(h1) {
  font-size: 1.8em;
  border-bottom: 1px solid #eaecef;
  padding-bottom: 0.2em;
}

.markdown-body :deep(h2) {
  font-size: 1.4em;
  border-bottom: 1px solid #eaecef;
  padding-bottom: 0.2em;
}

.markdown-body :deep(h3) {
  font-size: 1.2em;
}

.markdown-body :deep(p) {
  margin-top: 0;
  margin-bottom: 8px;
  line-height: 1.4;
}

.markdown-body :deep(ul),
.markdown-body :deep(ol) {
  padding-left: 1.5em;
  margin-top: 4px;
  margin-bottom: 8px;
}

.markdown-body :deep(li) {
  margin-top: 2px;
  margin-bottom: 2px;
  line-height: 1.4;
}

.markdown-body :deep(li + li) {
  margin-top: 4px;
}

.markdown-body :deep(blockquote) {
  padding: 0.5em 1em;
  color: #6a737d;
  border-left: 0.25em solid #dfe2e5;
  margin: 8px 0;
}

.markdown-body :deep(blockquote p) {
  margin: 0;
}

.markdown-body :deep(pre) {
  padding: 12px;
  margin: 8px 0;
  line-height: 1.4;
  background-color: #f6f8fa;
  border-radius: 3px;
  overflow: auto;
}

.markdown-body :deep(pre code) {
  background-color: transparent;
  padding: 0;
  margin: 0;
  font-size: 85%;
  line-height: inherit;
}

.markdown-body :deep(code) {
  padding: 0.2em 0.4em;
  margin: 0;
  font-size: 85%;
  background-color: rgba(27, 31, 35, 0.05);
  border-radius: 3px;
}

.markdown-body :deep(table) {
  margin: 8px 0;
  border-collapse: collapse;
  width: 100%;
}

.markdown-body :deep(table th),
.markdown-body :deep(table td) {
  padding: 4px 8px;
  border: 1px solid #dfe2e5;
}

.markdown-body :deep(table tr) {
  background-color: #fff;
  border-top: 1px solid #dfe2e5;
}

.markdown-body :deep(table tr:nth-child(2n)) {
  background-color: #f6f8fa;
}

/* 处理连续元素之间的间距 */
.markdown-body :deep(p + p) {
  margin-top: 4px;
}

.markdown-body :deep(p + ul),
.markdown-body :deep(p + ol),
.markdown-body :deep(ul + p),
.markdown-body :deep(ol + p) {
  margin-top: 4px;
}

.markdown-body :deep(h1 + p),
.markdown-body :deep(h2 + p),
.markdown-body :deep(h3 + p) {
  margin-top: 8px;
}

.markdown-body :deep(br) {
  display: block;
  margin: 4px 0;
  content: "";
}

.markdown-body :deep(hr) {
  margin: 12px 0;
  border: 0;
  border-top: 1px solid #eaecef;
}

.analysis-dialog :deep(.el-dialog) {
  background: rgba(30, 35, 45, 0.95);
  border: 1px solid rgba(64, 158, 255, 0.2);
  border-radius: 16px;
  box-shadow: 0 0 20px rgba(64, 158, 255, 0.2);
  backdrop-filter: blur(10px);
}

.analysis-dialog :deep(.el-dialog__header) {
  display: none;
}

.analysis-dialog :deep(.el-dialog__body) {
  padding: 30px;
}

.analysis-content {
  position: relative;
  color: #fff;
  text-align: center;
  padding: 20px;
  z-index: 1;
}

.analysis-header {
  margin-bottom: 20px;
  position: relative;
  z-index: 2;
}

.analysis-title {
  font-size: 24px;
  font-weight: 600;
  margin-bottom: 8px;
  background: linear-gradient(120deg, cornflowerblue, #85a5ff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.analysis-subtitle {
  color: rgba(255, 255, 255, 0.7);
  font-size: 14px;
}

.progress-container {
  position: relative;
  margin: 20px auto;
  max-width: 220px;
  z-index: 2;
  padding-top: 24px;
}

.progress-text {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
  font-size: 14px;
  font-weight: 600;
  color: cornflowerblue;
  background: rgba(255, 255, 255, 0.9);
  padding: 2px 8px;
  border-radius: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.analysis-progress {
  position: relative;
  z-index: 1;
}

.analysis-progress :deep(.el-progress-bar__outer) {
  background-color: rgba(100, 149, 237, 0.1) !important;
  border-radius: 4px;
  overflow: hidden;
}

.analysis-progress :deep(.el-progress-bar__inner) {
  background: linear-gradient(90deg, cornflowerblue, #85a5ff);
  box-shadow: 0 0 10px rgba(100, 149, 237, 0.5);
  border-radius: 4px;
  transition: width 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}

.analysis-steps {
  margin: 30px auto;
  max-width: 260px;
  position: relative;
  z-index: 2;
}

.step-item {
  display: flex;
  align-items: flex-start;
  margin: 12px 0;
  opacity: 0.5;
  transform: translateX(-10px);
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
}

.step-item.step-done {
  opacity: 1;
  transform: translateX(0);
}

.step-indicator {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-right: 12px;
}

.step-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background-color: cornflowerblue;
  box-shadow: 0 0 10px rgba(100, 149, 237, 0.5);
  transition: all 0.3s ease;
  transform: scale(0.8);
}

.step-item.step-done .step-dot {
  transform: scale(1);
  background-color: #85a5ff;
  box-shadow: 0 0 15px rgba(100, 149, 237, 0.8);
}

.step-line {
  width: 2px;
  height: 20px;
  background: rgba(100, 149, 237, 0.3);
  margin: 4px 0;
  transform: scaleY(0);
  transition: transform 0.4s ease;
  transform-origin: top center;
}

.analysis-decoration {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 280px;
  height: 280px;
  pointer-events: none;
  overflow: hidden;
  z-index: 0;
}

.tech-grid {
  position: absolute;
  top: -50%;
  left: -50%;
  width: 200%;
  height: 200%;
  background-image: 
    linear-gradient(var(--el-border-color-light) 1px, transparent 1px),
    linear-gradient(90deg, var(--el-border-color-light) 1px, transparent 1px);
  background-size: 20px 20px;
  transform: perspective(500px) rotateX(45deg);
  animation: gridMove 20s linear infinite;
  opacity: 0.5;
}

.tech-particles {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
}

.particle {
  position: absolute;
  width: 4px;
  height: 4px;
  background: rgba(100, 149, 237, 0.3);
  border-radius: 50%;
  animation: particleFloat 3s ease-in-out infinite;
}

.particle:nth-child(1) { left: 20%; top: 30%; animation-delay: 0s; }
.particle:nth-child(2) { left: 80%; top: 20%; animation-delay: 0.5s; }
.particle:nth-child(3) { left: 40%; top: 70%; animation-delay: 1s; }
.particle:nth-child(4) { left: 60%; top: 50%; animation-delay: 1.5s; }
.particle:nth-child(5) { left: 30%; top: 40%; animation-delay: 2s; }

.tech-lines {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  opacity: 0.15;
}

.tech-line {
  position: absolute;
  width: 100%;
  height: 1px;
  background: linear-gradient(90deg, 
    transparent 0%, 
    rgba(100, 149, 237, 0.4) 50%,
    transparent 100%
  );
  transform-origin: left center;
  animation: moveLine 3s cubic-bezier(0.4, 0, 0.2, 1) infinite;
}

.tech-line:nth-child(1) { top: 20%; animation-delay: 0s; }
.tech-line:nth-child(2) { top: 40%; animation-delay: 0.6s; }
.tech-line:nth-child(3) { top: 60%; animation-delay: 1.2s; }
.tech-line:nth-child(4) { top: 80%; animation-delay: 1.8s; }
.tech-line:nth-child(5) { top: 100%; animation-delay: 2.4s; }

@keyframes gridMove {
  0% {
    transform: perspective(500px) rotateX(45deg) translateY(0);
  }
  100% {
    transform: perspective(500px) rotateX(45deg) translateY(20px);
  }
}

@keyframes particleFloat {
  0%, 100% {
    transform: translateY(0) scale(1);
    opacity: 0.3;
  }
  50% {
    transform: translateY(-20px) scale(1.2);
    opacity: 0.6;
  }
}

@keyframes moveLine {
  0% {
    transform: translateX(-100%) scaleX(0.5);
    opacity: 0;
  }
  50% {
    transform: translateX(0%) scaleX(1);
    opacity: 1;
  }
  100% {
    transform: translateX(100%) scaleX(0.5);
    opacity: 0;
  }
}

@keyframes progressShine {
  0% {
    transform: translateX(-100%);
  }
  100% {
    transform: translateX(100%);
  }
}

/* 添加进入和离开动画 */
.analysis-dialog :deep(.el-dialog) {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.analysis-dialog :deep(.el-dialog).dialog-fade-enter-active {
  transform: scale(0.9);
  opacity: 0;
}

.analysis-dialog :deep(.el-dialog).dialog-fade-enter-to {
  transform: scale(1);
  opacity: 1;
}

.analysis-dialog :deep(.el-dialog).dialog-fade-leave-active {
  transform: scale(1.1);
  opacity: 0;
}

.spectrum-container {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 200px;
  height: 60px;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 6px;
}

.spectrum-bar {
  width: 4px;
  height: 20px;
  background: var(--el-color-primary-light-5);
  border-radius: 2px;
  animation: spectrumDance 1.2s ease-in-out infinite;
  transform-origin: center bottom;
}

.spectrum-bar:nth-child(1) { animation-delay: -1.2s; }
.spectrum-bar:nth-child(2) { animation-delay: -1.1s; }
.spectrum-bar:nth-child(3) { animation-delay: -1.0s; }
.spectrum-bar:nth-child(4) { animation-delay: -0.9s; }
.spectrum-bar:nth-child(5) { animation-delay: -0.8s; }
.spectrum-bar:nth-child(6) { animation-delay: -0.7s; }
.spectrum-bar:nth-child(7) { animation-delay: -0.6s; }
.spectrum-bar:nth-child(8) { animation-delay: -0.5s; }
.spectrum-bar:nth-child(9) { animation-delay: -0.4s; }
.spectrum-bar:nth-child(10) { animation-delay: -0.3s; }
.spectrum-bar:nth-child(11) { animation-delay: -0.2s; }
.spectrum-bar:nth-child(12) { animation-delay: -0.1s; }

@keyframes spectrumDance {
  0%, 100% {
    transform: scaleY(0.6);
    background: var(--el-color-primary-light-7);
  }
  25% {
    transform: scaleY(1);
    background: var(--el-color-primary-light-5);
  }
  50% {
    transform: scaleY(2.5);
    background: var(--el-color-primary-light-3);
  }
  75% {
    transform: scaleY(1.2);
    background: var(--el-color-primary-light-5);
  }
}

.guide-button {
  position: absolute;
  right: 20px;
  top: 20px;
  width: 36px;
  height: 36px;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--el-color-info-light-9);
  border: 1px solid var(--el-color-info-light-5);
  transition: all 0.3s ease;
}

.guide-button:hover {
  transform: scale(1.1);
  background: var(--el-color-info-light-8);
  border-color: var(--el-color-info-light-3);
}

.guide-button .el-icon {
  font-size: 18px;
  color: var(--el-color-info);
}
</style>
