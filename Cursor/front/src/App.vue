<template>
  <div class="app-shell">
    <div class="bg-deco"></div>

    <header class="header">
      <h1>LMY的智能工具</h1>
      <p>AI 恋爱大师 / AI 超级智能体</p>
    </header>

    <main class="main-card">
      <div class="tabs">
        <button
          :class="['tab-btn', activeTab === 'love' ? 'active' : '']"
          @click="switchTab('love')"
        >
          AI 恋爱大师
        </button>
        <button
          :class="['tab-btn', activeTab === 'manus' ? 'active' : '']"
          @click="switchTab('manus')"
        >
          AI 超级智能体
        </button>
      </div>

      <section v-if="activeTab === 'love'" class="panel">
        <div class="toolbar">
          <span class="chip">当前会话ID：{{ currentLoveChatId }}</span>
          <button class="btn ghost" @click="newLoveChat">新建恋爱会话</button>
        </div>

        <div class="inputs one-col">
          <input
            v-model="loveMessage"
            placeholder="输入发送给恋爱大师的内容..."
            @keyup.enter="sendLove"
          />
        </div>

        <div class="actions">
          <button class="btn primary" :disabled="loadingLove" @click="sendLove">
            {{ loadingLove ? '生成中...' : '发送' }}
          </button>
          <button class="btn ghost" @click="clearLove">清空输出</button>
        </div>

        <div class="output"><pre>{{ loveOutput }}</pre></div>
      </section>

      <section v-else class="panel">
        <div class="inputs one-col">
          <input
            v-model="manusMessage"
            placeholder="输入发送给超级智能体的任务..."
            @keyup.enter="sendManus"
          />
        </div>

        <div class="actions">
          <button class="btn primary" :disabled="loadingManus" @click="sendManus">
            {{ loadingManus ? '执行中...' : '发送' }}
          </button>
          <button class="btn ghost" @click="clearManus">清空输出</button>
        </div>

        <div class="output"><pre>{{ manusOutput }}</pre></div>
      </section>
    </main>
  </div>
</template>

<script setup>
import { onBeforeUnmount, ref } from 'vue'

const activeTab = ref('love')

const loveMessage = ref('')
const loveOutput = ref('')
const loadingLove = ref(false)
const currentLoveChatId = ref(createChatId())
let loveSource = null

const manusMessage = ref('')
const manusOutput = ref('')
const loadingManus = ref(false)
let manusSource = null

function createChatId() {
  if (typeof crypto !== 'undefined' && crypto.randomUUID) {
    return crypto.randomUUID()
  }
  return `chat-${Date.now()}-${Math.random().toString(36).slice(2, 10)}`
}

function switchTab(tab) {
  activeTab.value = tab
}

function newLoveChat() {
  closeLove()
  currentLoveChatId.value = createChatId()
  loveOutput.value = ''
  loveMessage.value = ''
}

function closeLove() {
  if (loveSource) {
    loveSource.close()
    loveSource = null
  }
  loadingLove.value = false
}

function closeManus() {
  if (manusSource) {
    manusSource.close()
    manusSource = null
  }
  loadingManus.value = false
}

function sendLove() {
  if (!loveMessage.value.trim()) return
  closeLove()
  loveOutput.value = ''
  loadingLove.value = true

  const params = new URLSearchParams({
    message: loveMessage.value,
    chatId: currentLoveChatId.value
  })

  loveSource = new EventSource(`/api/ai/love_app/chat/sse?${params.toString()}`)

  loveSource.onmessage = (e) => {
    loveOutput.value += e.data
  }

  loveSource.onerror = () => {
    closeLove()
  }
}

function sendManus() {
  if (!manusMessage.value.trim()) return
  closeManus()
  manusOutput.value = ''
  loadingManus.value = true

  const params = new URLSearchParams({
    message: manusMessage.value
  })

  manusSource = new EventSource(`/api/ai/manus/chat?${params.toString()}`)

  manusSource.onmessage = (e) => {
    manusOutput.value += e.data
  }

  manusSource.onerror = () => {
    closeManus()
  }
}

function clearLove() {
  closeLove()
  loveOutput.value = ''
}

function clearManus() {
  closeManus()
  manusOutput.value = ''
}

onBeforeUnmount(() => {
  closeLove()
  closeManus()
})
</script>
