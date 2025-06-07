<script setup>
/*eslint no-unused-vars: ["error", { "caughtErrors": "all", "caughtErrorsIgnorePattern": "^ignore" }]*/

import { ref, onMounted, watch, nextTick } from 'vue'
import { LettaClient } from '@letta-ai/letta-client'
import { marked } from 'marked'

// Types
/** @typedef {{ id: string, name: string, description?: string }} Agent */
/** @typedef {{ id: string, label: string, value: string }} MemoryBlock */
/** @typedef {{ id: string, messageType: string, content?: string, reasoning?: string, toolCall?: object, toolReturn?: object }} Message */
/** @typedef {{ id: string, text: string }} Passage */

const letta = new LettaClient({
  baseUrl: '/',
})

const markdownOptions = {
  gfm: true,
  breaks: true,
}

// State
const version = ref('') // string
const agents = ref([]) // Array<Agent>
const selectedAgent = ref('') // string
const contextWindow = ref(null)
const memoryBlocks = ref([]) // Array<MemoryBlock>
const passages = ref([]) // Array<Passage>
const messages = ref([]) // Array<Message>
const newMessage = ref('') // string
const error = ref(null) // string
const isLoading = ref({
  agents: false,
  messages: false,
  memory: false,
})

// Handle agent selection
watch(selectedAgent, async (newAgentId) => {
  contextWindow.value = null
  memoryBlocks.value = []
  passages.value = []
  messages.value = []

  if (!newAgentId) return

  isLoading.value.messages = true
  try {
    // Load all values in parallel
    const [contextWindowData, blocks, passagesData, messagesData] = await Promise.all([
      letta.agents.context.retrieve(newAgentId).catch(() => null),
      letta.agents.blocks.list(newAgentId),
      letta.agents.passages.list(newAgentId),
      letta.agents.messages.list(newAgentId),
    ])

    contextWindow.value = contextWindowData
    memoryBlocks.value = blocks.sort((a, b) => a.id.localeCompare(b.id))
    passages.value = passagesData
    messages.value = messagesData
    // wait until DOM is updated, then trigger jQuery change event on referenced element
    nextTick(() => {
      document.getElementById('message-new').scrollIntoView()
    })
  } catch (err) {
    error.value = `Failed to load agent data: ${err.message}`
    console.error(err)
  } finally {
    isLoading.value.messages = false
  }
})

// Fetch agents on mount
onMounted(async () => {
  isLoading.value.agents = true
  try {
    version.value = (await letta.health.check()).version
    agents.value = await letta.agents.list()
  } catch (err) {
    error.value = `Failed to load agents: ${err.message}`
    console.error(err)
  } finally {
    isLoading.value.agents = false
  }
})

// Send message
const sendMessage = async () => {
  if (!newMessage.value.trim() || !selectedAgent.value) return

  try {
    const msg = newMessage.value
    newMessage.value = ''

    // Add user message
    messages.value.push({
      id: Date.now().toString(),
      messageType: 'user_message',
      content: msg,
    })

    // Stream response
    const response = await letta.agents.messages.createStream(selectedAgent.value, {
      messages: [
        {
          role: 'user',
          content: [{ type: 'text', text: msg }],
        },
      ],
      // streamTokens: true,
    })

    let reloadMemory = false
    for await (const item of response) {
      messages.value.push(item)
      if (item.messageType === 'tool_return_message' && item.status === 'OK') reloadMemory = true
    }
    if (reloadMemory) memoryBlocks.value = await letta.agents.blocks.list(selectedAgent.value)
    contextWindow.value = await letta.agents.context.retrieve(selectedAgent.value).catch(() => null)
  } catch (err) {
    error.value = `Failed to send message: ${err.message}`
    console.error(err)
  }
}

const formatJSON = (str) => {
  try {
    return JSON.stringify(JSON.parse(str), null, 2)
  } catch (ignore) {
    return ''
  }
}

// Handle Enter key
const handleEnter = (event) => {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    sendMessage()
  }
}
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="./assets/logo.svg" width="125" height="125" />

    <div class="header-container">
      Letta {{ version }}

      <div class="agent-select">
        <h3>Agent</h3>
        <div v-if="isLoading.agents" class="loading">Loading…</div>
        <select id="agents" v-model="selectedAgent" v-if="!isLoading.agents">
          <option value="">Select an agent:</option>
          <option v-for="agent in agents" :key="agent.id" :value="agent.id">
            {{ agent.name }}
          </option>
        </select>
      </div>

      <div v-if="error" class="error-message">
        {{ error }}
      </div>

      <div class="agent-select" v-if="contextWindow !== null">
        <h3>Context</h3>
        <div class="progress-container">
          <div
            class="progress-bar"
            :style="{
              width: `${(contextWindow.contextWindowSizeCurrent / contextWindow.contextWindowSizeMax) * 100}%`,
            }"
          ></div>
          <div class="progress-label">
            {{ contextWindow.contextWindowSizeCurrent }} /
            {{ contextWindow.contextWindowSizeMax }}
          </div>
        </div>
      </div>

      <div v-if="memoryBlocks.length > 0" class="memories core-memory">
        <h3>Core Memory</h3>
        <ul>
          <li v-for="block in memoryBlocks" :key="block.id">
            <strong>{{ block.label }}</strong
            >: {{ block.value }}
          </li>
        </ul>
      </div>

      <div v-if="passages.length > 0" class="memories passages">
        <h3>Memories</h3>
        <ul>
          <li v-for="block in passages" :key="block.id">
            {{ block.text }}
          </li>
        </ul>
      </div>
    </div>
  </header>

  <main>
    <div id="messages" class="message-list">
      <div class="message" v-for="message in messages" :key="message.id">
        <div class="message-icon" :title="message.messageType">
          <span v-if="message.messageType === 'user_message'">👤</span>
          <span v-else-if="message.messageType === 'assistant_message'">🤖</span>
          <span v-else-if="message.messageType === 'reasoning_message'">🧠</span>
          <span v-else-if="message.messageType === 'tool_call_message'">📤</span>
          <span v-else-if="message.messageType === 'tool_return_message'">📥</span>
          <span v-else>❓</span>
        </div>
        <div class="message-bubble" :class="message.messageType">
          <div v-if="message.content" v-html="marked.parse(message.content, markdownOptions)"></div>
          <div
            v-else-if="message.reasoning"
            v-html="marked.parse(message.reasoning, markdownOptions)"
          ></div>
          <div v-else-if="message.toolCall">
            <strong>{{ message.toolCall.name }}</strong>
            <pre>{{ formatJSON(message.toolCall.arguments) }}</pre>
          </div>
          <div v-else-if="message.toolReturn">
            <span v-if="message.status === 'success'">✅</span>
            <span v-else>❌</span>
            <strong>{{ message.name }}</strong>
            {{ message.status }}
            <pre v-if="message.status !== 'success'">{{ message.toolReturn }}</pre>
          </div>
          <div v-else>
            <pre><code>{{ JSON.stringify(message, null, 2) }}</code></pre>
          </div>
        </div>
      </div>
    </div>

    <div id="message-new" class="message-bubble user_message">
      <textarea
        v-model="newMessage"
        @keydown.enter="handleEnter"
        placeholder="Type a message..."
        :disabled="isLoading.messages || !selectedAgent"
      ></textarea>
    </div>

    <div v-if="isLoading.messages" class="loading">Loading messages...</div>
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 20px;
  overflow-y: auto;
  box-sizing: border-box;
}

main {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding: 20px;
  overflow-y: auto;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

.header-container {
  display: flex;
  flex-direction: column;
  gap: 15px;
  padding: 20px;
  flex-shrink: 0;
}

.agent-select {
  padding: 10px 12px;
  border: 1px solid #555;
  border-radius: 8px;
  background: #1e1e1e;
  color: #fff;
  font-size: 1rem;
  transition: border-color 0.3s;
}

.agent-select:focus {
  border-color: #007bff;
  outline: none;
}

/* Progress Bar Styles */
.progress-container {
  position: relative;
  width: 100%;
  background-color: #eee;
  border-radius: 8px;
  overflow: hidden;
  margin-top: 10px;
  height: 20px;
}

.progress-bar {
  position: absolute;
  left: 0;
  top: 0;
  height: 20px;
  background-color: #007bff;
  border-radius: 8px;
  transition: width 0.3s ease;
  z-index: 1;
}

.progress-label {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  top: 50%;
  transform: translate(-50%, -50%);
  color: black;
  font-size: 12px;
  z-index: 2;
}

.memories {
  margin-top: 20px;
  padding: 15px;
  border: 1px solid #444;
  border-radius: 8px;
  background-color: #1e1e1e;
}

.memories ul {
  list-style-type: none;
  padding-left: 0;
}

.memories li {
  margin-bottom: 10px;
  padding: 8px;
  background-color: #2d2d2d;
  border-left: 4px solid #007bff;
  border-radius: 4px;
  overflow: hidden;
  max-height: 7.5em;
  transition:
    max-height 0.3s ease,
    overflow 0.3s ease;
}

.memories li:hover {
  max-height: none;
  overflow: visible;
  border-left: 4px solid #3399ff; /* lighter blue on hover */
}

.memories strong {
  color: #007bff;
  font-weight: bold;
}

.message-list {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 15px;
  padding: 15px;
}

.message {
  display: flex;
  align-items: flex-start;
  margin-bottom: 15px;
}

.message-icon {
  margin-right: 12px;
  font-size: 1.4em;
  flex-shrink: 0;
  cursor: help;
}

.message-bubble {
  background-color: #f1f1f1;
  padding: 12px 16px;
  border-radius: 12px;
  max-width: 80%;
  word-wrap: break-word;
  color: #000;
  position: relative;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.message-bubble.user_message {
  background-color: #d1e7dd;
  align-self: flex-end;
  margin-left: auto;
  margin-right: 0;
}

.message-bubble.reasoning_message {
  background-color: #495057;
  color: #fff;
}

.message-bubble.tool_call_message,
.message-bubble.tool_return_message {
  background-color: #495057;
  color: #fff;
}

pre {
  white-space: pre-wrap;
  word-wrap: break-word;
  margin: 10px 0 0;
  background: #2d2d2d;
  padding: 10px;
  border-radius: 6px;
  color: #ccc;
}

#message-new {
  flex: 0 0 auto; /* Prevent from growing too much */
  padding: 10px;
}

#message-new textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid #ccc;
  border-radius: 10px;
  resize: none;
  font-family: inherit;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.3s;
}

#message-new textarea:focus {
  border-color: #007bff;
}

.loading {
  color: #666;
  text-align: center;
  padding: 10px;
}

.error-message {
  color: #dc3545;
  background: #f8d7da;
  padding: 10px;
  border-radius: 6px;
  margin-top: 10px;
}

@media (min-width: 1024px) {
  header {
    display: flex;
  }

  .logo {
    margin: 0 2rem 0 0;
  }
}
</style>
