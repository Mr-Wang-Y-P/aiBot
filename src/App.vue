<template>
  <div class="app-container" :class="{ 'dark-mode': darkMode }">
    <div class="app-header">
      <div class="logo-container">
        <div class="logo">
          <div class="logo-icon">
            <BrainCircuitIcon class="brain-icon" />
          </div>
          <h1>Wang Chat</h1>
        </div>
        <button class="theme-toggle" @click="toggleDarkMode">
          <SunIcon v-if="darkMode" />
          <MoonIcon v-else />
        </button>
      </div>
      <div class="model-selector">
        <label for="model-select">选择 AI 模型:</label>
        <select id="model-select" v-model="selectedModel" :disabled="isLoading">
          <option v-for="model in models" :key="model.id" :value="model.id">
            {{ model.name }}
          </option>
        </select>
      </div>
      <div class="model-selector">
        <label for="model-input">输入ApiKey:</label>
        <input id="model-input" type="text" v-model="inputApiKey" />
        </div>
    </div>

    <div class="chat-container" ref="chatContainer">
      <div class="messages-container">
        <div v-if="messages.length === 0" class="welcome-screen">
          <div class="welcome-animation">
            <div class="ai-avatar">
              <BrainCircuitIcon class="brain-icon large" />
            </div>
          </div>
          <h2>欢迎使用 AI 聊天助手</h2>
          <p>选择一个 AI 模型并开始对话吧！</p>
          <div class="feature-list">
            <div class="feature">
              <MessageSquareIcon />
              <span>实时对话</span>
            </div>
            <!-- <div class="feature">
              <FileUpIcon />
              <span>文件上传</span>
            </div> -->
            <div class="feature">
              <SlidersIcon />
              <span>多模型支持</span>
            </div>
          </div>
        </div>

        <div v-else class="message-list">
          <div
            v-for="(message, index) in messages"
            :key="index"
            class="message"
            :class="message.role"
          >
            <div class="message-avatar">
              <UserIcon v-if="message.role === 'user'" />
              <BrainCircuitIcon v-else />
            </div>
            <div class="message-content">
              <div class="message-text" v-html="formatMessage(message.content)"></div>
              <div class="message-time">{{ formatTime(message.timestamp) }}</div>
            </div>
          </div>
        </div>

        <div v-if="isLoading" class="typing-indicator">
          <div class="dot"></div>
          <div class="dot"></div>
          <div class="dot"></div>
        </div>
      </div>

      <div class="input-container">
        <!-- <div class="file-upload">
          <label for="file-input" class="file-upload-label">
            <PaperclipIcon />
          </label>
          <input
            id="file-input"
            type="file"
            @change="handleFileUpload"
            :disabled="isLoading || isUploading"
          />
          <div v-if="uploadedFile" class="uploaded-file">
            <FileIcon /> {{ uploadedFile.originalname }}
            <button class="remove-file" @click="removeFile">
              <XIcon />
            </button>
          </div>
          <div v-if="isUploading" class="upload-progress">
            上传中...
          </div>
        </div> -->

        <div class="message-input-wrapper">
          <textarea
            v-model="userInput"
            class="message-input"
            placeholder="输入消息..."
            @keydown.enter.prevent="sendMessage"
            :disabled="isLoading"
            ref="messageInput"
            rows="1"
          ></textarea>
          <button
            class="send-button"
            @click="sendMessage"
            :disabled="!canSendMessage"
          >
            <SendIcon />
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import { 
  BrainCircuitIcon, 
  UserIcon, 
  SendIcon, 
  PaperclipIcon, 
  FileIcon, 
  XIcon,
  MessageSquareIcon,
  FileUpIcon,
  SlidersIcon,
  SunIcon,
  MoonIcon
} from 'lucide-vue-next'
import { marked } from 'marked'
import hljs from 'highlight.js'

// 状态管理
const darkMode = ref(window.matchMedia('(prefers-color-scheme: dark)').matches)
const messages = ref([])
const userInput = ref('')
const isLoading = ref(false)
const selectedModel = ref('')
const models = ref([])
const chatContainer = ref(null)
const messageInput = ref(null)
const uploadedFile = ref(null)
const isUploading = ref(false)
const inputApiKey = ref('')
// 计算属性
const canSendMessage = computed(() => {
  return userInput.value.trim().length > 0 && !isLoading.value && selectedModel.value
})

// 方法
const toggleDarkMode = () => {
  darkMode.value = !darkMode.value
}

const formatTime = (timestamp) => {
  const date = new Date(timestamp)
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
}

const formatMessage = (content) => {
  // 配置 marked 使用 highlight.js
  marked.setOptions({
    highlight: function(code, lang) {
      if (lang && hljs.getLanguage(lang)) {
        return hljs.highlight(code, { language: lang }).value
      }
      return hljs.highlightAuto(code).value
    },
    breaks: true
  })
  
  return marked(content)
}

const scrollToBottom = async () => {
  await nextTick()
  if (chatContainer.value) {
    chatContainer.value.scrollTop = chatContainer.value.scrollHeight
  }
}

const fetchModels = async () => {
  try {
    const response = await fetch('http://localhost:3100/api/models')
    if (response.ok) {
      models.value = await response.json()
      if (models.value.length > 0) {
        selectedModel.value = models.value[0].id
        if(localStorage.getItem(selectedModel.value)){
          inputApiKey.value = localStorage.getItem(selectedModel.value)
        }
        models.value?.forEach(model => {
          if (localStorage.getItem(selectedModel.value)) {
            inputApiKey.value = localStorage.getItem(selectedModel.value)
          }else {
            localStorage.setItem(model?.id,'')
          }
        });
      }
    } else {
      console.error('获取模型列表失败')
    }
  } catch (error) {
    console.error('获取模型时出错:', error)
  }
}

const handleFileUpload = async (event) => {
  const file = event.target.files[0]
  if (!file) return
  
  isUploading.value = true
  
  try {
    const formData = new FormData()
    formData.append('file', file)
    
    const response = await fetch('http://localhost:3100/api/upload', {
      method: 'POST',
      body: formData
    })
    
    if (response.ok) {
      const result = await response.json()
      uploadedFile.value = result.file
      
      // 添加文件信息到用户输入
      if (userInput.value) {
        userInput.value += `\n\n[文件: ${file.name}]`
      } else {
        userInput.value = `[文件: ${file.name}]`
      }
    } else {
      console.error('文件上传失败')
    }
  } catch (error) {
    console.error('上传文件时出错:', error)
  } finally {
    isUploading.value = false
  }
}

const removeFile = () => {
  uploadedFile.value = null
  // 从用户输入中移除文件信息
  userInput.value = userInput.value.replace(/\n\n\[文件: .*\]|\[文件: .*\]/, '')
}

const sendMessage = async () => {
  if (!canSendMessage.value) return
  
  const userMessage = userInput.value.trim()
  userInput.value = ''
  
  // 添加用户消息到聊天
  messages.value.push({
    role: 'user',
    content: userMessage,
    timestamp: Date.now()
  })
  
  await scrollToBottom()
  
  // 准备发送到AI的消息
  const messagesToSend = messages.value.map(msg => ({
    role: msg.role,
    content: msg.content
  }))
  
  // 添加AI消息占位符
  messages.value.push({
    role: 'assistant',
    content: '',
    timestamp: Date.now()
  })
  
  isLoading.value = true
  
  try {
    const response = await fetch('http://localhost:3100/api/chat', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        modelId: selectedModel.value,
        inputApiKey: inputApiKey.value,
        messages: messagesToSend
      })
    })
    
    if (!response.ok) {
      throw new Error(`HTTP error! Status: ${response.status}`)
    }
    
    const reader = response.body.getReader()
    const decoder = new TextDecoder()
    
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      
      const chunk = decoder.decode(value, { stream: true })
      const lines = chunk.split('\n\n')
      
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6).trim()
          if (data === '[DONE]') break
          
          try {
            const parsed = JSON.parse(data)
            // 从服务器接收到的数据现在应该只包含 content 字段
            if (parsed.content) {
              messages.value[messages.value.length - 1].content += parsed.content
              await scrollToBottom()
              localStorage.setItem(selectedModel.value, inputApiKey.value)
            }
          } catch (e) {
            console.error('解析JSON失败:', e)
          }
        }
      }
    }
  } catch (error) {
    console.error('发送消息时出错:', error)
    // 添加错误消息
    messages.value[messages.value.length - 1].content = '抱歉，发生了错误。请稍后再试。'
  } finally {
    isLoading.value = false
    uploadedFile.value = null
    await scrollToBottom()
    messageInput.value.focus()
  }
}

// 生命周期钩子
onMounted(() => {
  fetchModels()
  messageInput.value.focus()
  
  // 自动调整文本区域高度
  const adjustTextareaHeight = () => {
    messageInput.value.style.height = 'auto'
    messageInput.value.style.height = `${messageInput.value.scrollHeight}px`
  }
  
  watch(userInput, () => {
    nextTick(adjustTextareaHeight)
  })
  watch(selectedModel,()=> {
    if(localStorage.getItem(selectedModel.value)){
      inputApiKey.value = localStorage.getItem(selectedModel.value)
    }
  })
})
</script>

<style>
@import 'highlight.js/styles/atom-one-dark.css';

:root {
  --primary-color: #0070f3;
  --primary-hover: #0761d1;
  --background-light: #ffffff;
  --background-dark: #111111;
  --text-light: #333333;
  --text-dark: #f0f0f0;
  --border-light: #e0e0e0;
  --border-dark: #333333;
  --message-user-bg-light: #f0f7ff;
  --message-user-bg-dark: #1a365d;
  --message-ai-bg-light: #f5f5f5;
  --message-ai-bg-dark: #2d3748;
  --input-bg-light: #ffffff;
  --input-bg-dark: #1e1e1e;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
}

body {
  background-color: var(--background-light);
  color: var(--text-light);
}

.app-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  /* max-width: 1200px; */
  margin: 0 auto;
  background-color: var(--background-light);
  color: var(--text-light);
  transition: all 0.3s ease;
}

.app-container.dark-mode {
  background-color: var(--background-dark);
  color: var(--text-dark);
}

.app-header {
  padding: 1rem;
  border-bottom: 1px solid var(--border-light);
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.dark-mode .app-header {
  border-bottom: 1px solid var(--border-dark);
}

.logo-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.logo {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.logo-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 10px;
  background: linear-gradient(135deg, var(--primary-color), #6d28d9);
  color: white;
}

.brain-icon {
  width: 24px;
  height: 24px;
}

.brain-icon.large {
  width: 48px;
  height: 48px;
}

.theme-toggle {
  background: none;
  border: none;
  cursor: pointer;
  color: inherit;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  transition: background-color 0.3s;
}

.theme-toggle:hover {
  background-color: rgba(0, 0, 0, 0.05);
}

.dark-mode .theme-toggle:hover {
  background-color: rgba(255, 255, 255, 0.1);
}

.model-selector {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.model-selector label {
  font-weight: 500;
}

.model-selector select {
  padding: 0.5rem 1rem;
  border-radius: 8px;
  border: 1px solid var(--border-light);
  background-color: var(--input-bg-light);
  color: var(--text-light);
  font-size: 1rem;
  flex-grow: 1;
  cursor: pointer;
  transition: all 0.3s ease;
}
.model-selector input {
  padding: 0.5rem 1rem;
  border-radius: 8px;
  border: 1px solid var(--border-light);
  background-color: var(--input-bg-light);
  color: var(--text-light);
  font-size: 1rem;
  flex-grow: 1;
  cursor: pointer;
  transition: all 0.3s ease;
}
.dark-mode .model-selector select {
  border: 1px solid var(--border-dark);
  background-color: var(--input-bg-dark);
  color: var(--text-dark);
}

.chat-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  scroll-behavior: smooth;
}

.welcome-screen {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  text-align: center;
  padding: 2rem;
}

.welcome-animation {
  margin-bottom: 2rem;
}

.ai-avatar {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  background: linear-gradient(135deg, var(--primary-color), #6d28d9);
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto;
  box-shadow: 0 10px 25px rgba(0, 112, 243, 0.2);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(0, 112, 243, 0.4);
  }
  70% {
    box-shadow: 0 0 0 20px rgba(0, 112, 243, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(0, 112, 243, 0);
  }
}

.welcome-screen h2 {
  font-size: 2rem;
  margin-bottom: 1rem;
  background: linear-gradient(135deg, var(--primary-color), #6d28d9);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.welcome-screen p {
  font-size: 1.2rem;
  margin-bottom: 2rem;
  color: #666;
}

.dark-mode .welcome-screen p {
  color: #aaa;
}

.feature-list {
  display: flex;
  gap: 2rem;
  margin-top: 2rem;
}

.feature {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.5rem;
}

.feature svg {
  width: 32px;
  height: 32px;
  color: var(--primary-color);
}

.message-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.message {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  border-radius: 12px;
  animation: fadeIn 0.3s ease-in-out;
  max-width: 85%;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.message.user {
  background-color: var(--message-user-bg-light);
  align-self: flex-end;
  border-bottom-right-radius: 0;
}

.dark-mode .message.user {
  background-color: var(--message-user-bg-dark);
}

.message.assistant {
  background-color: var(--message-ai-bg-light);
  align-self: flex-start;
  border-bottom-left-radius: 0;
}

.dark-mode .message.assistant {
  background-color: var(--message-ai-bg-dark);
}

.message-avatar {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: var(--primary-color);
  color: white;
  flex-shrink: 0;
}

.message.user .message-avatar {
  background-color: #6d28d9;
}

.message-content {
  flex: 1;
  min-width: 0;
}

.message-text {
  word-wrap: break-word;
  line-height: 1.5;
}

.message-text pre {
  margin: 1rem 0;
  padding: 1rem;
  border-radius: 8px;
  overflow-x: auto;
  background-color: #282c34;
  color: #abb2bf;
}

.message-text code {
  font-family: 'Fira Code', monospace;
  font-size: 0.9rem;
}

.message-text p {
  margin-bottom: 0.75rem;
}

.message-text p:last-child {
  margin-bottom: 0;
}

.message-time {
  font-size: 0.75rem;
  color: #666;
  margin-top: 0.5rem;
  text-align: right;
}

.dark-mode .message-time {
  color: #aaa;
}

.typing-indicator {
  display: flex;
  align-items: center;
  gap: 0.25rem;
  padding: 1rem;
  background-color: var(--message-ai-bg-light);
  border-radius: 12px;
  width: fit-content;
  margin-top: 1rem;
}

.dark-mode .typing-indicator {
  background-color: var(--message-ai-bg-dark);
}

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background-color: #666;
  animation: bounce 1.5s infinite;
}

.dark-mode .dot {
  background-color: #aaa;
}

.dot:nth-child(2) {
  animation-delay: 0.2s;
}

.dot:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes bounce {
  0%, 60%, 100% {
    transform: translateY(0);
  }
  30% {
    transform: translateY(-6px);
  }
}

.input-container {
  padding: 1rem;
  border-top: 1px solid var(--border-light);
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.dark-mode .input-container {
  border-top: 1px solid var(--border-dark);
}

.file-upload {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.file-upload-label {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: transparent;
  color: #666;
  cursor: pointer;
  transition: all 0.3s ease;
}

.dark-mode .file-upload-label {
  color: #aaa;
}

.file-upload-label:hover {
  background-color: rgba(0, 0, 0, 0.05);
  color: var(--primary-color);
}

.dark-mode .file-upload-label:hover {
  background-color: rgba(255, 255, 255, 0.1);
}

#file-input {
  display: none;
}

.uploaded-file {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.5rem;
  background-color: rgba(0, 0, 0, 0.05);
  border-radius: 8px;
  font-size: 0.9rem;
}

.dark-mode .uploaded-file {
  background-color: rgba(255, 255, 255, 0.1);
}

.remove-file {
  background: none;
  border: none;
  cursor: pointer;
  color: #666;
  display: flex;
  align-items: center;
  justify-content: center;
}

.dark-mode .remove-file {
  color: #aaa;
}

.upload-progress {
  font-size: 0.9rem;
  color: #666;
}

.dark-mode .upload-progress {
  color: #aaa;
}

.message-input-wrapper {
  display: flex;
  align-items: flex-end;
  gap: 0.5rem;
  background-color: var(--input-bg-light);
  border: 1px solid var(--border-light);
  border-radius: 12px;
  padding: 0.5rem;
  transition: all 0.3s ease;
}

.dark-mode .message-input-wrapper {
  background-color: var(--input-bg-dark);
  border: 1px solid var(--border-dark);
}

.message-input-wrapper:focus-within {
  border-color: var(--primary-color);
  box-shadow: 0 0 0 2px rgba(0, 112, 243, 0.2);
}

.message-input {
  flex: 1;
  border: none;
  background: transparent;
  font-size: 1rem;
  resize: none;
  max-height: 150px;
  outline: none;
  color: var(--text-light);
  line-height: 1.5;
}

.dark-mode .message-input {
  color: var(--text-dark);
}

.message-input::placeholder {
  color: #999;
}

.send-button {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: var(--primary-color);
  color: white;
  border: none;
  cursor: pointer;
  transition: all 0.3s ease;
}

.send-button:hover {
  background-color: var(--primary-hover);
}

.send-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.dark-mode .send-button:disabled {
  background-color: #444;
}

@media (max-width: 768px) {
  .feature-list {
    flex-direction: column;
    gap: 1rem;
  }
  
  .message {
    max-width: 95%;
  }
}
</style>

## 项目说明

这个项目创建了一个功能齐全的 AI 聊天机器人应用，具有以下特点：

1. **炫酷的用户界面**：
   - 支持亮色/暗色模式
   - 流畅的动画效果
   - 响应式设计，适配各种设备
   - 代码高亮显示

2. **多模型支持**：
   - 可以选择不同的 AI 模型（OpenAI、Anthropic Claude、DeepSeek 等）
   - 每个模型使用不同的 API 密钥

3. **文件上传功能**：
   - 支持上传文件并在对话中引用
   - 显示上传进度和文件信息

4. **实时流式响应**：
   - AI 回复实时流式显示，提供更好的用户体验

5. **Markdown 支持**：
   - AI 回复支持 Markdown 格式，包括代码块高亮

## 使用说明

1. 在后端项目根目录创建 `.env` 文件，添加以下内容：