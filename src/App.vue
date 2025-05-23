<script setup>
import { ref, onMounted } from 'vue'
import { LettaClient } from '@letta-ai/letta-client'

const letta = new LettaClient({
  baseUrl: '/api/',
})

const agents = ref([])

onMounted(async () => {
  try {
    agents.value = await letta.agents.list()
  } catch (error) {
    console.error('Error fetching agents:', error)
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
    <select id="agents">
      <option value="">Select an option</option>
      <option v-for="option in agents" :key="option.id">{{ option.name }}</option>
    </select>
    <ul id="messages">
      <li>Message 1</li>
      <li>Message 2</li>
      <li>Message 3</li>
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
