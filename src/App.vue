<script lang="ts" setup>
import { ref, computed } from 'vue'
import { GoogleGenerativeAI } from '@google/generative-ai'

// 定義聊天歷史紀錄的資料結構
interface ChatRecord {
  id: string
  conversationId: string
  prompt: string
  response: string
  model: string
  timestamp: Date
}

// store Gemini prompt
const prompt = ref('')
// store Gemini result
const result = ref('')
// store displayed result for typewriter effect
const displayedResult = ref('')
// loading state
const loading = ref(false)
// typing state for typewriter effect
const isTyping = ref(false)
// store error message
const error = ref<string | null>(null)
// store selected model
const selectedModel = ref('gemini-2.5-flash')
// store chat history (for sidebar)
const chatHistory = ref<ChatRecord[]>([])
// store search keyword for history
const searchKeyword = ref('')
// store currently selected chat from history
const selectedChatId = ref<string | null>(null)
// store current conversation ID
const currentConversationId = ref<string | null>(null)
// store current conversation messages for context
const currentConversation = ref<ChatRecord[]>([])

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

// Computed property to check if in an active conversation
const isInActiveConversation = computed(() => {
  return currentConversationId.value !== null && currentConversation.value.length > 0
})

// --- 函式定義 ---

// Typewriter effect function
function typewriterEffect(text: string, speed: number = 30) {
  return new Promise<void>((resolve) => {
    isTyping.value = true
    displayedResult.value = ''
    let index = 0

    const typeInterval = setInterval(() => {
      if (index < text.length) {
        displayedResult.value += text.charAt(index)
        index++
      } else {
        clearInterval(typeInterval)
        isTyping.value = false
        resolve()
      }
    }, speed)
  })
}

// Function to clear current conversation and start new
function clearCurrentConversation() {
  prompt.value = ''
  result.value = ''
  displayedResult.value = ''
  error.value = null
  selectedChatId.value = null
  currentConversationId.value = null
  currentConversation.value = []
}

// Function to select a chat from history
async function selectChatFromHistory(chatId: string) {
  const chat = chatHistory.value.find((c) => c.id === chatId)
  if (chat) {
    selectedChatId.value = chatId
    result.value = chat.response
    prompt.value = ''
    error.value = null

    // Load the entire conversation thread
    currentConversationId.value = chat.conversationId
    currentConversation.value = chatHistory.value
      .filter((c) => c.conversationId === chat.conversationId)
      .sort((a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime())

    // Apply typewriter effect to show the selected response
    await typewriterEffect(chat.response)
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
  displayedResult.value = ''
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
    // 3. Create conversation ID if this is a new conversation
    if (!currentConversationId.value) {
      currentConversationId.value = Date.now().toString()
    }

    // 4. Build conversation history for context
    const conversationHistory = []
    for (const record of currentConversation.value) {
      conversationHistory.push({ role: 'user', parts: [{ text: record.prompt }] })
      conversationHistory.push({ role: 'model', parts: [{ text: record.response }] })
    }

    // 5. use Google GenAI SDK to call Gemini API with conversation history
    const model = genAI.getGenerativeModel({ model: selectedModel.value })
    const chat = model.startChat({
      history: conversationHistory,
    })

    const response = await chat.sendMessage(currentPrompt)

    // 6. after getting response, store the result
    const responseText = response.response.text()
    result.value = responseText

    // 7. Stop loading and start typewriter effect
    loading.value = false
    await typewriterEffect(responseText)

    // 8. Add to current conversation and chat history
    const chatRecord: ChatRecord = {
      id: Date.now().toString(),
      conversationId: currentConversationId.value,
      prompt: currentPrompt,
      response: responseText,
      model: selectedModel.value,
      timestamp: new Date(),
    }

    currentConversation.value.push(chatRecord)
    chatHistory.value.unshift(chatRecord) // Add to beginning of array

    // 9. Clear current prompt for next question
    prompt.value = ''
  } catch (err) {
    // 10. if there's an error, log it and show a friendly message
    console.error('API 呼叫失敗:', err) // show detailed error in console
    error.value = '發生錯誤，請檢查 API Key 或網路連線。' // show user-friendly message
    loading.value = false
    isTyping.value = false
  }
}
</script>

<template>
  <div class="bg-gray-100 text-gray-800 font-sans min-h-screen">
    <div class="container mx-auto p-4 flex flex-col md:flex-row gap-4 max-w-7xl">
      <!-- Left Sidebar - Chat History -->
      <div
        class="w-full md:w-1/3 bg-white rounded-lg shadow-md p-4 flex flex-col order-2 md:order-1"
      >
        <h2 class="text-xl font-bold text-gray-800 mb-4">對話歷史</h2>

        <!-- Search Bar -->
        <div class="mb-4">
          <input
            v-model="searchKeyword"
            type="text"
            placeholder="搜尋對話內容..."
            class="w-full p-2 text-sm border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none"
          />
        </div>

        <!-- New Chat Button -->
        <button
          @click="clearCurrentConversation"
          class="mb-4 py-2 px-4 text-sm text-white bg-green-600 rounded-md hover:bg-green-700 transition-colors"
        >
          + 新對話
        </button>

        <!-- Chat History List -->
        <div class="flex-1 overflow-y-auto">
          <div
            v-if="filteredChatHistory.length === 0"
            class="text-gray-500 text-sm text-center py-8"
          >
            {{ searchKeyword ? '找不到相關對話' : '還沒有對話紀錄' }}
          </div>
          <div
            v-for="chat in filteredChatHistory"
            :key="chat.id"
            @click="selectChatFromHistory(chat.id)"
            :class="[
              'mb-2 p-3 border rounded-lg cursor-pointer transition-colors hover:bg-blue-50',
              selectedChatId === chat.id ? 'border-blue-500 bg-blue-50' : 'border-gray-200',
            ]"
          >
            <div class="text-sm font-medium text-gray-800 mb-1 overflow-hidden">
              {{ chat.prompt.length > 40 ? chat.prompt.substring(0, 40) + '...' : chat.prompt }}
            </div>
            <div class="text-xs text-gray-500 flex justify-between items-center">
              <span class="truncate mr-2">{{ chat.model }}</span>
              <div class="flex items-center gap-2 flex-shrink-0">
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

      <!-- Main Chat Area -->
      <div class="flex-1 bg-white rounded-lg shadow-md p-6 order-1 md:order-2 flex flex-col">
        <!-- Header -->
        <div class="mb-6">
          <h1 class="text-3xl font-bold text-blue-600 mb-2">Gemini API</h1>
          <p class="text-gray-600">這是一個簡單的應用，讓你體驗在 Vue 中串接 AI 服務有多麼容易。</p>
        </div>

        <!-- Model Selection -->
        <div class="mb-6">
          <label for="model-select" class="block text-sm font-medium text-gray-700 mb-2">
            選擇 AI 模型：
          </label>
          <select
            id="model-select"
            v-model="selectedModel"
            :disabled="loading || isTyping"
            class="w-full p-3 text-base border border-gray-300 rounded-md focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition bg-white disabled:bg-gray-100 disabled:cursor-not-allowed"
          >
            <option v-for="model in availableModels" :key="model.value" :value="model.value">
              {{ model.label }}
            </option>
          </select>
        </div>

        <!-- Chat Messages Area -->
        <div
          class="flex-1 mb-6 overflow-y-auto max-h-96 border border-gray-200 rounded-lg p-4 bg-gray-50"
        >
          <div v-if="currentConversation.length === 0" class="text-gray-500 text-center py-8">
            開始新的對話吧！
          </div>

          <!-- Messages -->
          <div v-for="(message, index) in currentConversation" :key="index" class="mb-4">
            <!-- User Message -->
            <div class="flex justify-end mb-2">
              <div class="max-w-[80%] bg-blue-600 text-white p-3 rounded-lg rounded-br-none">
                <div class="whitespace-pre-wrap break-words">{{ message.prompt }}</div>
                <div class="text-xs text-blue-100 mt-1">
                  {{ new Date(message.timestamp).toLocaleTimeString() }}
                </div>
              </div>
            </div>

            <!-- AI Response -->
            <div class="flex justify-start">
              <div class="max-w-[80%] bg-white border p-3 rounded-lg rounded-bl-none">
                <div class="whitespace-pre-wrap break-words text-gray-800">
                  {{
                    index === currentConversation.length - 1 && isTyping
                      ? displayedResult
                      : message.response
                  }}
                </div>
                <div
                  v-if="index === currentConversation.length - 1 && isTyping"
                  class="inline-block w-2 h-4 bg-blue-600 animate-pulse ml-1"
                ></div>
                <div class="text-xs text-gray-500 mt-1 flex justify-between">
                  <span>{{ message.model }}</span>
                  <span>{{ new Date(message.timestamp).toLocaleTimeString() }}</span>
                </div>
              </div>
            </div>
          </div>

          <!-- Loading Indicator -->
          <div v-if="loading" class="flex justify-start mb-4">
            <div class="bg-white border p-3 rounded-lg rounded-bl-none">
              <div class="flex items-center space-x-2">
                <div class="w-2 h-2 bg-gray-400 rounded-full animate-bounce"></div>
                <div
                  class="w-2 h-2 bg-gray-400 rounded-full animate-bounce"
                  style="animation-delay: 0.1s"
                ></div>
                <div
                  class="w-2 h-2 bg-gray-400 rounded-full animate-bounce"
                  style="animation-delay: 0.2s"
                ></div>
                <span class="text-gray-500 text-sm ml-2">AI 正在思考...</span>
              </div>
            </div>
          </div>
        </div>

        <!-- Error Display -->
        <div v-if="error" class="mb-4 p-4 bg-red-100 border border-red-300 text-red-700 rounded-lg">
          <p><strong>糟糕！</strong> {{ error }}</p>
        </div>

        <!-- Input Area -->
        <div class="border-t pt-4">
          <div class="flex flex-col sm:flex-row gap-3">
            <textarea
              v-model="prompt"
              rows="3"
              :placeholder="
                isInActiveConversation
                  ? '繼續對話或詢問相關問題...'
                  : '在這裡輸入任何你想問 Gemini 的問題...'
              "
              :disabled="loading || isTyping"
              class="flex-1 p-3 text-base border border-gray-300 rounded-md resize-y focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition disabled:bg-gray-100 disabled:cursor-not-allowed"
            ></textarea>
            <button
              @click="callGeminiAPI"
              :disabled="loading || isTyping || !prompt.trim()"
              class="px-6 py-3 text-white bg-blue-600 rounded-md cursor-pointer transition-colors duration-300 hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed whitespace-nowrap"
            >
              {{ loading ? '思考中...' : isTyping ? '正在顯示回答...' : '傳送' }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
