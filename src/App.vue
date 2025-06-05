<script setup>
import { ref, onMounted, watch } from 'vue'
import { LettaClient } from '@letta-ai/letta-client'
import { marked } from 'marked'

const letta = new LettaClient({
  baseUrl: '/',
})

const markdownOptions = {
  gfm: true,
  breaks: true,
}

const agents = ref([])
const selectedAgent = ref('')
const memoryBlocks = ref([])
const messages = ref([])
const newMessage = ref('')

const showModalWindow = ref(false)
const modalTitle = ref('')
const modalText = ref('')

const showModal = async (type, name) => {
  showModalWindow.value = true
  if (type === 'core') {
    modalTitle.value = 'Core memory: ' + name
    modalText.value = '…'
    modalText.value = (await letta.agents.blocks.retrieve(selectedAgent.value, name)).value
  }
}

const closeModal = () => {
  showModalWindow.value = false
}

onMounted(async () => {
  try {
    agents.value = await letta.agents.list()
  } catch (error) {
    console.error('Error fetching agents:', error)
  }
})

watch(selectedAgent, async (newAgentId) => {
  if (newAgentId) {
    try {
      memoryBlocks.value = await letta.agents.blocks.list(selectedAgent.value)
      messages.value = await letta.agents.messages.list(newAgentId)
    } catch (error) {
      console.error('Error fetching messages:', error)
    }
  }
})

const sendMessage = async () => {
  if (!newMessage.value.trim() || !selectedAgent.value) return

  try {
    const msg = newMessage.value
    newMessage.value = ''
    messages.value.push({
      messageType: 'user_message',
      content: msg,
    })
    const response = await letta.agents.messages.createStream(selectedAgent.value, {
      messages: [
        {
          role: 'user',
          content: [
            {
              type: 'text',
              text: msg,
            },
          ],
        },
      ],
      // stream_tokens: true,
    })
    for await (const item of response) {
      messages.value.push(item)
    }
  } catch (error) {
    console.error('Error sending message:', error)
  }
}

// Handle Enter key press to send message
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
      <HelloWorld msg="You did it!" />
      <div>
        <select id="agents" v-model="selectedAgent">
          <option value="">Select an agent:</option>
          <option v-for="agent in agents" :key="agent.id" :value="agent.id">
            {{ agent.name }}
          </option>
        </select>
      </div>
      <div v-if="memoryBlocks.length > 0">
        Core memory:
        <button
          v-for="block in memoryBlocks"
          :key="block.id"
          @click="showModal('core', block.label)"
        >
          {{ block.label }}
        </button>
      </div>
      <div v-if="showModalWindow" class="modal">
        <div class="modal-content">
          <span class="close" @click="closeModal">&times;</span>
          <h2>{{ modalTitle }}</h2>
          <p>{{ modalText }}</p>
        </div>
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
      ></textarea>
    </div>
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 200px;
  padding: 20px;
  box-sizing: border-box;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

.modal {
  position: fixed;
  z-index: 1000;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background-color: #333;
  color: #fff;
  padding: 20px;
  border-radius: 8px;
  position: relative;
  max-width: 500px;
  width: 90%;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.3);
}

.modal-content h2 {
  margin-top: 0;
}

.close {
  position: absolute;
  top: 10px;
  right: 15px;
  font-size: 24px;
  cursor: pointer;
  color: #fff;
}

.modal-content p {
  margin: 10px 0 0;
}

#messages {
  max-width: 70vw;
}

.message {
  position: relative;
  display: flex;
  align-items: flex-start;
  margin-bottom: 10px;
}

.message-icon {
  margin-right: 10px;
  font-size: 1.2em;
  flex-shrink: 0;
  cursor: help; /* Optional: to indicate it's a tooltip */
}

.message-bubble {
  background-color: #f1f1f1;
  padding: 10px 15px;
  border-radius: 10px;
  max-width: 70%;
  word-wrap: break-word;
  color: #000; /* Dark text on light background */
  overflow-x: auto;
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
}

textarea {
  width: 100%;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 8px;
  resize: none;
  font-family: inherit;
  font-size: 1rem;
  outline: none;
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
