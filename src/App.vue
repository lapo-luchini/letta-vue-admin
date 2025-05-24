<script setup>
import { ref, onMounted, watch } from 'vue'
import { LettaClient } from '@letta-ai/letta-client'
import { marked } from 'marked'

const letta = new LettaClient({
  baseUrl: '/api/',
})

const markdownOptions = {
  gfm: true,
  breaks: true,
}

const agents = ref([])
const selectedAgent = ref(null)
const messages = ref([])

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
      messages.value = await letta.agents.messages.list(newAgentId)
    } catch (error) {
      console.error('Error fetching messages:', error)
    }
  }
})
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="./assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="You did it!" />
    </div>
  </header>

  <main>
    <select id="agents" v-model="selectedAgent">
      <option value="">Select an option</option>
      <option v-for="agent in agents" :key="agent.id" :value="agent.id">{{ agent.name }}</option>
    </select>
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
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
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
