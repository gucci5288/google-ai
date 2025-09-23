<script lang="ts" setup>
import { ref, computed } from 'vue'
import { GoogleGenerativeAI } from '@google/generative-ai'

// 定義聊天歷史紀錄的資料結構
interface ChatRecord {
  id: string
  prompt: string
  response: string
  model: string
  timestamp: Date
}

// store Gemini prompt
const prompt = ref('')
// store Gemini result
const result = ref('')
// loading state
const loading = ref(false)
// store error message
const error = ref<string | null>(null)
// store selected model
const selectedModel = ref('gemini-2.5-flash')
// store chat history
const chatHistory = ref<ChatRecord[]>([])
// store search keyword for history
const searchKeyword = ref('')
// store currently selected chat from history
const selectedChatId = ref<string | null>(null)

// available Gemini models
const availableModels = [
  { value: 'gemini-2.5-flash', label: 'Gemini 2.5 Flash' },
  { value: 'gemini-2.5-pro', label: 'Gemini 2.5 Pro' },
]

// read API key from environment variable
const API_KEY = import.meta.env.VITE_GEMINI_API_KEY

const genAI = new GoogleGenerativeAI(API_KEY)

// Computed property for filtered chat history based on search
const filteredChatHistory = computed(() => {
  if (!searchKeyword.value.trim()) {
    return chatHistory.value
  }
  return chatHistory.value.filter(
    (chat) =>
      chat.prompt.toLowerCase().includes(searchKeyword.value.toLowerCase()) ||
      chat.response.toLowerCase().includes(searchKeyword.value.toLowerCase()),
  )
})

// --- 函式定義 ---

// Function to clear current conversation and start new
function clearCurrentConversation() {
  prompt.value = ''
  result.value = ''
  error.value = null
  selectedChatId.value = null
}

// Function to select a chat from history
function selectChatFromHistory(chatId: string) {
  const chat = chatHistory.value.find((c) => c.id === chatId)
  if (chat) {
    selectedChatId.value = chatId
    result.value = chat.response
    // Don't populate prompt to avoid confusion
    prompt.value = ''
    error.value = null
  }
}

// Function to delete a chat from history
function deleteChatFromHistory(chatId: string) {
  const index = chatHistory.value.findIndex((c) => c.id === chatId)
  if (index > -1) {
    chatHistory.value.splice(index, 1)
    if (selectedChatId.value === chatId) {
      clearCurrentConversation()
    }
  }
}

// 這是我們最核心的函式，用來呼叫 Gemini API
// async/await 是處理非同步操作（例如網路請求）的現代語法
async function callGeminiAPI() {
  // 1. begin by resetting states
  loading.value = true
  result.value = ''
  error.value = null

  // 2. check if prompt is empty
  if (!prompt.value.trim()) {
    error.value = '請輸入你的問題！'
    loading.value = false
    return // 結束函式
  }

  // Store the current prompt before clearing
  const currentPrompt = prompt.value

  try {
    // 3. use Google GenAI SDK to call Gemini API
    const model = genAI.getGenerativeModel({ model: selectedModel.value })
    const response = await model.generateContent(currentPrompt)

    // 4. after getting response, store the result
    // based on Google GenAI SDK response structure
    const responseText = response.response.text()
    result.value = responseText

    // 5. Add to chat history
    const chatRecord: ChatRecord = {
      id: Date.now().toString(),
      prompt: currentPrompt,
      response: responseText,
      model: selectedModel.value,
      timestamp: new Date(),
    }
    chatHistory.value.unshift(chatRecord) // Add to beginning of array

    // 6. Clear current prompt for next question
    prompt.value = ''
  } catch (err) {
    // 7. if there's an error, log it and show a friendly message
    console.error('API 呼叫失敗:', err) // show detailed error in console
    error.value = '發生錯誤，請檢查 API Key 或網路連線。' // show user-friendly message
  } finally {
    // 8. no matter success or failure, stop loading state
    loading.value = false
  }
}
</script>

<template>
  <div
    class="bg-gray-100 text-gray-800 font-sans flex flex-col md:flex-row p-1 md:p-2 min-h-screen gap-2 md:gap-0"
  >
    <!-- Left Sidebar - Chat History -->
    <div
      class="w-full md:w-1/3 bg-white rounded-lg shadow-md md:mr-4 p-2 md:p-4 flex flex-col order-2 md:order-1"
    >
      <h2 class="text-lg md:text-xl font-bold text-gray-800 mb-2 md:mb-4">對話歷史</h2>

      <!-- Search Bar -->
      <div class="mb-2 md:mb-4">
        <input
          v-model="searchKeyword"
          type="text"
          placeholder="搜尋對話內容..."
          class="w-full p-1.5 md:p-2 text-xs md:text-sm border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none"
        />
      </div>

      <!-- New Chat Button -->
      <button
        @click="clearCurrentConversation"
        class="mb-2 md:mb-4 py-1.5 md:py-2 px-3 md:px-4 text-xs md:text-sm text-white bg-green-600 rounded-md hover:bg-green-700 transition-colors"
      >
        + 新對話
      </button>

      <!-- Chat History List -->
      <div class="flex-1 overflow-y-auto max-h-40 md:max-h-none">
        <div
          v-if="filteredChatHistory.length === 0"
          class="text-gray-500 text-xs md:text-sm text-center py-4 md:py-8"
        >
          {{ searchKeyword ? '找不到相關對話' : '還沒有對話紀錄' }}
        </div>
        <div
          v-for="chat in filteredChatHistory"
          :key="chat.id"
          @click="selectChatFromHistory(chat.id)"
          :class="[
            'mb-1 md:mb-2 p-2 md:p-3 border rounded-lg cursor-pointer transition-colors hover:bg-blue-50',
            selectedChatId === chat.id ? 'border-blue-500 bg-blue-50' : 'border-gray-200',
          ]"
        >
          <div class="text-xs md:text-sm font-medium text-gray-800 mb-1 overflow-hidden">
            {{ chat.prompt.length > 40 ? chat.prompt.substring(0, 40) + '...' : chat.prompt }}
          </div>
          <div class="text-xs md:text-xs text-gray-500 flex justify-between items-center">
            <span class="truncate mr-2">{{ chat.model }}</span>
            <div class="flex items-center gap-1 md:gap-2 flex-shrink-0">
              <span class="text-xs">{{ new Date(chat.timestamp).toLocaleDateString() }}</span>
              <button
                @click.stop="deleteChatFromHistory(chat.id)"
                class="text-red-500 hover:text-red-700 text-xs w-4 h-4 flex items-center justify-center"
                title="刪除"
              >
                ✕
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Right Main Area - Current Conversation -->
    <div class="flex-1 bg-white rounded-lg shadow-md p-3 md:p-8 order-1 md:order-2">
      <h1 class="text-2xl md:text-3xl font-bold text-blue-600 mb-1 md:mb-2">Gemini API</h1>
      <p class="text-gray-600 mb-4 md:mb-8 text-sm md:text-base">
        這是一個簡單的應用，讓你體驗在 Vue 中串接 AI 服務有多麼容易。
      </p>

      <!-- Model Selection -->
      <div class="form-group flex flex-col mb-3 md:mb-6">
        <label for="model-select" class="text-xs md:text-sm font-medium text-gray-700 mb-1 md:mb-2">
          選擇 AI 模型：
        </label>
        <select
          id="model-select"
          v-model="selectedModel"
          class="w-full p-2 md:p-3 text-sm md:text-base border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition bg-white"
        >
          <option v-for="model in availableModels" :key="model.value" :value="model.value">
            {{ model.label }}
          </option>
        </select>
      </div>

      <div class="form-group flex flex-col">
        <textarea
          v-model="prompt"
          rows="3"
          placeholder="在這裡輸入任何你想問 Gemini 的問題..."
          class="w-full p-2 md:p-3 text-sm md:text-base border border-gray-300 rounded-md mb-2 md:mb-4 resize-y focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition"
        ></textarea>
        <button
          @click="callGeminiAPI"
          :disabled="loading"
          class="py-2 md:py-3 px-4 md:px-5 text-sm md:text-base text-white bg-blue-600 rounded-md cursor-pointer transition-colors duration-300 hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed"
        >
          {{ loading ? '思考中...' : '傳送問題' }}
        </button>
      </div>

      <div v-if="loading" class="flex justify-center my-4 md:my-8">
        <div
          class="w-8 h-8 md:w-10 md:h-10 border-4 border-gray-200 border-t-blue-600 rounded-full animate-spin"
        ></div>
      </div>

      <div
        v-if="error"
        class="mt-3 md:mt-4 p-3 md:p-4 bg-red-100 border border-red-300 text-red-700 rounded-lg"
      >
        <p class="text-sm md:text-base"><strong>糟糕！</strong> {{ error }}</p>
      </div>

      <div
        v-if="result"
        class="mt-4 md:mt-8 p-3 md:p-6 bg-gray-50 border border-gray-200 rounded-lg"
      >
        <h2 class="mt-0 text-lg md:text-xl font-semibold text-gray-800 mb-2 md:mb-4">
          Gemini 的回答：
        </h2>
        <pre class="whitespace-pre-wrap break-words font-mono leading-relaxed text-xs md:text-sm">{{
          result
        }}</pre>
      </div>
    </div>
  </div>
</template>
