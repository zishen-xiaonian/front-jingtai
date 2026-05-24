<script setup>
import { computed } from 'vue'
import '../style.css'

const props = defineProps({
  user: {
    type: Object,
    default: null,
  },
})

const emotionEvolutionParts = computed(() => {
  const value = String(props.user?.emotionEvolution || '').trim()

  if (!value) {
    return []
  }

  return value.split(/\s*->\s*/).filter(Boolean)
})
</script>

<template>
  <section class="module-block emotion-evolution-analysis-panel">
    <div class="block-head emotion-evolution-head">
      <h3>情绪演变分析</h3>
    </div>

    <div class="emotion-evolution-content">
      <div class="emotion-evolution-section">
        <p class="emotion-evolution-label">用户情绪演变过程</p>
        <p class="emotion-evolution-value emotion-evolution-flow">
          <template v-if="emotionEvolutionParts.length">
            <template v-for="(item, index) in emotionEvolutionParts" :key="`${item}-${index}`">
              <span class="emotion-evolution-step">{{ item }}</span>
              <span
                v-if="index < emotionEvolutionParts.length - 1"
                class="emotion-evolution-arrow"
                aria-hidden="true"
              ></span>
            </template>
          </template>
          <span v-else>-</span>
        </p>
      </div>

      <div class="emotion-evolution-section">
        <p class="emotion-evolution-label">依据</p>
        <p class="emotion-evolution-value emotion-evolution-basis">{{ props.user?.emotionBasis || '-' }}</p>
      </div>
    </div>
  </section>
</template>
