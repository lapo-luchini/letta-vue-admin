<script setup>
import { ref, onMounted, watch } from 'vue'
import { LettaClient } from '@letta-ai/letta-client'

const letta = new LettaClient({
  baseUrl: '/api/',
})

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
    <ul id="messages">
      <li v-for="message in messages" :key="message.id">{{ message.content }}</li>
    </ul>
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

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
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
