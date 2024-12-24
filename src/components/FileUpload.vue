<template>
  <div class="w-full">
    <div
      class="relative border-2 border-dashed border-gray-300 rounded-lg p-6 flex flex-col items-center justify-center w-[800px] h-[600px]"
      :class="{ 
        'border-primary bg-primary/5': isDragging,
        'opacity-50 cursor-not-allowed': uploadStatus.uploading 
      }"
      @dragover.prevent="handleDragOver"
      @dragleave.prevent="handleDragLeave"
      @drop.prevent="handleDrop"
      @paste.prevent="handlePaste"
      tabindex="0"
    >
      <div class="flex flex-col items-center gap-4">
        <!-- 上传进度条 -->
        <div
          v-if="uploadStatus.uploading"
          class="absolute top-0 left-0 right-0 h-2 bg-gray-200 overflow-hidden z-10"
        >
          <div
            class="h-full bg-blue-500 transition-all duration-300 ease-out"
            :style="{ width: `${uploadStatus.progress}%` }"
          />
        </div>

        <div class="w-12 h-12 text-gray-400">
          <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
          </svg>
        </div>
        <div class="text-center">
          <p class="text-sm text-gray-600">将文件拖到此处，或粘贴剪贴板图片，或</p>
          <label class="mt-2 cursor-pointer" :class="{ 'cursor-not-allowed': uploadStatus.uploading }">
            <span class="text-primary hover:underline">点击上传</span>
            <input
              type="file"
              class="hidden"
              accept=".jpg,.png"
              @change="handleFileSelect"
              :disabled="uploadStatus.uploading"
            >
          </label>
        </div>
        <p class="text-xs text-gray-500">
          只能上传jpg/png文件，且不超过2MB
        </p>
        <!-- 上传中状态显示 -->
        <div v-if="uploadStatus.uploading" class="text-lg font-semibold text-primary">
          正在上传... {{ uploadStatus.progress }}%
        </div>
      </div>
    </div>
    
    <!-- 错误提示 -->
    <div v-if="error" class="mt-2 text-sm text-red-500">
      {{ error }}
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, watch } from 'vue'

const isDragging = ref(false)
const error = ref('')
const uploadStatus = reactive({
  uploading: false,
  progress: 0
})

watch(() => uploadStatus.progress, (newValue) => {
  console.log('Progress updated:', newValue)
})

const uploadFile = async (file) => {
  try {
    uploadStatus.uploading = true
    uploadStatus.progress = 0
    
    const formData = new FormData()
    formData.append('file', file)
    
    const xhr = new XMLHttpRequest()
    
    // 创建一个Promise来处理上传
    const uploadPromise = new Promise((resolve, reject) => {
      xhr.upload.onprogress = (event) => {
        if (event.lengthComputable) {
          console.log('Upload progress:', event.loaded, event.total)
          // 模拟较慢的上传速度
          const actualProgress = Math.round((event.loaded / event.total) * 100)
          setTimeout(() => {
            uploadStatus.progress = actualProgress
          }, 500) // 添加500ms延迟
        }
      }
      
      xhr.upload.onloadstart = () => {
        console.log('Upload started')
      }
      
      xhr.upload.onloadend = () => {
        console.log('Upload finished')
      }

      xhr.onload = () => {
        console.log('Response received:', xhr.status)
        if (xhr.status === 200) {
          try {
            const response = JSON.parse(xhr.responseText)
            // 确保100%进度能显示一会
            setTimeout(() => {
              resolve(response)
            }, 300)
          } catch (err) {
            reject(new Error('解析响应失败'))
          }
        } else {
          reject(new Error('上传失败'))
        }
      }
      
      xhr.onerror = () => {
        console.error('XHR error occurred')
        reject(new Error('网络错误'))
      }
    })
    
    // 发送请求
    xhr.open('POST', '/upload')
    // 限制上传速度
    xhr.setRequestHeader('X-Requested-With', 'XMLHttpRequest')
    xhr.upload.onprogress = (event) => {
      if (event.lengthComputable) {
        const progress = Math.round((event.loaded / event.total) * 100)
        // 使用setTimeout来模拟较慢的上传速度
        setTimeout(() => {
          uploadStatus.progress = progress
        }, progress * 20) // 根据进度添加不同的延迟
      }
    }
    xhr.send(formData)
    
    // 等待上传完成
    const result = await uploadPromise
    error.value = ''
    return result
  } catch (err) {
    error.value = '文件上传失败：' + err.message
  } finally {
    uploadStatus.uploading = false
    uploadStatus.progress = 0
  }
}

const validateFile = (file) => {
  // 检查文件类型
  if (!['image/jpeg', 'image/png'].includes(file.type)) {
    error.value = '只支持jpg/png格式的图片'
    return false
  }
  
  // 检查文件大小
  if (file.size > 2 * 1024 * 1024) {
    error.value = '文件大小不能超过2MB'
    return false
  }
  
  error.value = ''
  return true
}

const handlePaste = async (e) => {
  const items = e.clipboardData.items
  let file = null

  for (let i = 0; i < items.length; i++) {
    if (items[i].type.indexOf('image') !== -1) {
      file = items[i].getAsFile()
      break
    }
  }

  if (file && validateFile(file)) {
    await uploadFile(file)
  } else if (!file) {
    error.value = '剪贴板中没有图片'
  }
}

const handleDragOver = () => {
  isDragging.value = true
}

const handleDragLeave = () => {
  isDragging.value = false
}

const handleDrop = async (e) => {
  isDragging.value = false
  const file = e.dataTransfer.files[0]
  if (file && validateFile(file)) {
    await uploadFile(file)
  }
}

const handleFileSelect = async (e) => {
  const file = e.target.files[0]
  if (file && validateFile(file)) {
    await uploadFile(file)
  }
}
</script>

<style scoped>
.bg-primary {
  background: linear-gradient(90deg, #0ea5e9, #0284c7);
  background-size: 200% 100%;
  animation: gradient 2s linear infinite;
}

@keyframes gradient {
  0% {
    background-position: 0% 0%;
  }
  100% {
    background-position: 200% 0%;
  }
}
</style> 