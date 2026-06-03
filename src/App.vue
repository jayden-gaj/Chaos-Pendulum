<script setup>
import { ref } from 'vue'
import ChaosPendulum from './components/ChaosPendulum.vue'
import DodgeGame from './components/DodgeGame.vue'
import InfoTab from './components/InfoTab.vue'

const tab = ref('sim')
</script>

<template>
  <nav class="tabs">
    <button :class="{ active: tab === 'sim' }" @click="tab = 'sim'">Simulator</button>
    <button :class="{ active: tab === 'game' }" @click="tab = 'game'">Dodge Game</button>
    <button :class="{ active: tab === 'info' }" @click="tab = 'info'">Info</button>
  </nav>
  <ChaosPendulum v-if="tab === 'sim'" />
  <DodgeGame v-else-if="tab === 'game'" />
  <InfoTab v-else />
  <footer class="footer">
    <div class="footer-center">
        <p style="font-weight: bold; font-size: 1.1rem;">AP Physics C Final Project 2026</p>
        <p>Ethan Hoffman</p>
        <p>Jayden Gajewski</p>
    </div>
  </footer>
</template>

<style scoped>
.tabs {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
  padding: 0.75rem 0.75rem 0;
}
.tabs button {
  background: #1f2937;
  color: #e5e7eb;
  border: 1px solid #374151;
  padding: 0.4rem 1rem;
  border-radius: 6px;
  font: inherit;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
}
.tabs button:hover {
  background: #2a3645;
}
.tabs button.active {
  background: #41b883;
  color: #0b0d12;
  border-color: transparent;
}
.footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1.5rem 2rem;
    margin-top: 3rem;
    background: #1f2937;
    border-top: 1px solid #ddd;
    color: #ffffff;
    font-size: 0.95rem;
}

.footer-center {
    flex: 1;
    text-align: center;
}

.footer-center p
{
    margin: 0.25rem 0;
}

</style>