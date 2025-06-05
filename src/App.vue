<script setup>
import { ref, onMounted, watch } from 'vue'
import { LettaClient } from '@letta-ai/letta-client'
import { marked } from 'marked'

// Types
/** @typedef {{ id: string, name: string, description?: string }} Agent */
/** @typedef {{ id: string, label: string, value: string }} MemoryBlock */
/** @typedef {{ id: string, messageType: string, content?: string, reasoning?: string, toolCall?: object, toolReturn?: object }} Message */

const letta = new LettaClient({
  baseUrl: '/',
})

const markdownOptions = {
  gfm: true,
  breaks: true,
}

// State
const agents = ref([]) // Array<Agent>
const selectedAgent = ref('') // string
const memoryBlocks = ref([]) // Array<MemoryBlock>
const messages = ref([]) // Array<Message>
const newMessage = ref('')
const error = ref(null)
const isLoading = ref({
  agents: false,
  messages: false,
  memory: false,
})

// Handle agent selection
watch(selectedAgent, async (newAgentId) => {
  if (!newAgentId) return

  isLoading.value.messages = true
  try {
    memoryBlocks.value = await letta.agents.blocks.list(newAgentId)
    messages.value = await letta.agents.messages.list(newAgentId)
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
      // stream_tokens: true,
    })

    for await (const item of response) {
      messages.value.push(item)
    }
  } catch (err) {
    error.value = `Failed to send message: ${err.message}`
    console.error(err)
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

    <div class="wrapper">
      <div>
        <select id="agents" v-model="selectedAgent" :disabled="isLoading.agents">
          <option value="">Select an agent:</option>
          <option v-for="agent in agents" :key="agent.id" :value="agent.id">
            {{ agent.name }}
          </option>
        </select>
      </div>

      <div v-if="error" class="error-message">
        {{ error }}
      </div>

      <div v-if="isLoading.agents" class="loading">Loading agents...</div>

      <div v-if="memoryBlocks.length > 0" class="core-memory">
        <h3>Core Memory</h3>
        <ul>
          <li v-for="block in memoryBlocks" :key="block.id">
            <strong>{{ block.label }}</strong
            >: {{ block.value }}
          </li>
        </ul>
      </div>
    </div>
  </header>

  <main>
    <div id="messages">
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
            <pre>{{ JSON.stringify(JSON.parse(message.toolCall.arguments), null, 2) }}</pre>
          </div>
          <div v-else-if="message.toolReturn">
            <strong>{{ message.name }}</strong>
            <pre>{{ JSON.stringify(JSON.parse(message.toolReturn), null, 2) }}</pre>
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
  min-width: 250px; /* Fixed width for smaller screens */
  max-width: 300px; /* Limit width on larger screens */
  padding: 20px;
  box-sizing: border-box;
  border-right: 1px solid #444;
  flex-shrink: 0; /* Prevent header from shrinking */
}

main {
  flex: 1; /* Take up remaining space */
  display: flex;
  flex-direction: column;
  padding: 20px;
  overflow: hidden;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

.core-memory {
  margin-top: 20px;
  padding: 15px;
  border: 1px solid #444;
  border-radius: 8px;
  background-color: #1e1e1e;
}

.core-memory ul {
  list-style-type: none;
  padding-left: 0;
}

.core-memory li {
  margin-bottom: 10px;
  padding: 8px;
  background-color: #2d2d2d;
  border-left: 4px solid #007bff;
  border-radius: 4px;
}

.core-memory strong {
  color: #007bff;
  font-weight: bold;
}

#messages {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 10px;
  overflow-y: auto;
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
  max-width: 70%;
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

  header .wrapper {
    display: flex;
    place-items: flex-start;
    flex-wrap: wrap;
  }
}
</style>
