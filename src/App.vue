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
  <main class="container">
    <h1>我的第一個 AI 應用 (Vue + Gemini)</h1>
    <p class="subtitle">這是一個簡單的應用，讓你體驗在 Vue 中串接 AI 服務有多麼容易。</p>

    <div class="form-group">
      <textarea
        v-model="prompt"
        rows="4"
        placeholder="在這裡輸入任何你想問 Gemini 的問題..."
      ></textarea>
      <button @click="callGeminiAPI" :disabled="loading">
        {{ loading ? '思考中...' : '傳送問題' }}
      </button>
    </div>

    <div v-if="loading" class="loading-spinner"></div>

    <div v-if="error" class="error-message">
      <p><strong>糟糕！</strong> {{ error }}</p>
    </div>

    <div v-if="result" class="result-card">
      <h2>Gemini 的回答：</h2>
      <pre>{{ result }}</pre>
    </div>
  </main>
</template>

<style>
/* 為了讓畫面好看一點，加一些簡單的 CSS */
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background-color: #f0f2f5;
  color: #333;
  display: flex;
  justify-content: center;
  padding-top: 50px;
}
.container {
  max-width: 700px;
  width: 100%;
  padding: 2rem;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}
h1 {
  font-size: 2rem;
  color: #1a73e8; /* Google Blue */
}
.subtitle {
  color: #5f6368;
  margin-bottom: 2rem;
}
.form-group {
  display: flex;
  flex-direction: column;
}
textarea {
  width: 100%;
  padding: 12px;
  font-size: 1rem;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-bottom: 1rem;
  resize: vertical;
}
button {
  padding: 12px 20px;
  font-size: 1rem;
  color: white;
  background-color: #1a73e8;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s;
}
button:hover {
  background-color: #185abc;
}
button:disabled {
  background-color: #9e9e9e;
  cursor: not-allowed;
}
.result-card {
  margin-top: 2rem;
  padding: 1.5rem;
  background-color: #f8f9fa;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
}
.result-card h2 {
  margin-top: 0;
  color: #3c4043;
}
pre {
  white-space: pre-wrap; /* 自動換行 */
  word-wrap: break-word; /* 斷詞 */
  font-family: 'Courier New', Courier, monospace;
  line-height: 1.6;
}
.error-message {
  margin-top: 1rem;
  padding: 1rem;
  background-color: #fce8e6;
  border: 1px solid #f9a8a2;
  color: #c5221f;
  border-radius: 8px;
}
.loading-spinner {
  margin: 2rem auto;
  border: 5px solid #f3f3f3;
  border-top: 5px solid #1a73e8;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
}
@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
