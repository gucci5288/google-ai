<script lang="ts" setup>
import { ref } from 'vue'
import axios from 'axios'

// store Gemini prompt
const prompt = ref('')
// store Gemini result
const result = ref('')
// loading state
const loading = ref(false)
// store error message
const error = ref<string | null>(null)

// read API key from environment variable
const API_KEY = import.meta.env.VITE_GEMINI_API_KEY

// Google Gemini Pro model URL
const API_URL_ADDRESS = import.meta.env.VITE_GOOGLE_GEMINI_API_URL

// Google Gemini Pro model API point
const API_URL = `${API_URL_ADDRESS}?key=${API_KEY}`

// --- 函式定義 ---

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

  try {
    // 3. use axios to send a POST request to Gemini API
    const response = await axios.post(API_URL, {
      // this is the request body, following Gemini API spec
      contents: [
        {
          parts: [
            {
              text: prompt.value,
            },
          ],
        },
      ],
    })

    // 4.after getting response, store the result
    // based on Gemini API response structure
    result.value = response.data.candidates[0].content.parts[0].text
  } catch (err) {
    // 5. if there's an error, log it and show a friendly message
    console.error('API 呼叫失敗:', err) // show detailed error in console
    error.value = '發生錯誤，請檢查 API Key 或網路連線。' // show user-friendly message
  } finally {
    // 6. no matter success or failure, stop loading state
    loading.value = false
  }
}
</script>

<template>
  <div class="bg-gray-100 text-gray-800 font-sans flex justify-center pt-12 min-h-screen">
    <main class="container bg-white max-w-3xl w-full p-8 rounded-lg shadow-md h-full">
      <h1 class="text-3xl font-bold text-blue-600">我的第一個 AI 應用 (Vue + Gemini)</h1>
      <p class="text-gray-600 mb-8">
        這是一個簡單的應用，讓你體驗在 Vue 中串接 AI 服務有多麼容易。
      </p>

      <div class="form-group flex flex-col">
        <textarea
          v-model="prompt"
          rows="4"
          placeholder="在這裡輸入任何你想問 Gemini 的問題..."
          class="w-full p-3 text-base border border-gray-300 rounded-md mb-4 resize-y focus:ring-2 focus:ring-blue-500 focus:border-transparent outline-none transition"
        ></textarea>
        <button
          @click="callGeminiAPI"
          :disabled="loading"
          class="py-3 px-5 text-base text-white bg-blue-600 rounded-md cursor-pointer transition-colors duration-300 hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed"
        >
          {{ loading ? '思考中...' : '傳送問題' }}
        </button>
      </div>

      <div v-if="loading" class="flex justify-center my-8">
        <div
          class="w-10 h-10 border-4 border-gray-200 border-t-blue-600 rounded-full animate-spin"
        ></div>
      </div>

      <div v-if="error" class="mt-4 p-4 bg-red-100 border border-red-300 text-red-700 rounded-lg">
        <p><strong>糟糕！</strong> {{ error }}</p>
      </div>

      <div v-if="result" class="mt-8 p-6 bg-gray-50 border border-gray-200 rounded-lg">
        <h2 class="mt-0 text-xl font-semibold text-gray-800 mb-4">Gemini 的回答：</h2>
        <pre class="whitespace-pre-wrap break-words font-mono leading-relaxed">{{ result }}</pre>
      </div>
    </main>
  </div>
</template>
