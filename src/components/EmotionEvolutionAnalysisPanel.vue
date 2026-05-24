<script setup>
import '../style.css'

const props = defineProps({
  user: {
    type: Object,
    default: null,
  },
})
</script>

<template>
  <section class="module-block emotion-evolution-analysis-panel">
    <div class="block-head">
      <h3>情绪演变分析</h3>
    </div>

    <div class="emotion-evolution-content">
      <div class="emotion-evolution-section">
        <p class="emotion-evolution-label">用户情绪演变过程</p>
        <p class="emotion-evolution-value">{{ props.user?.emotionEvolution || '-' }}</p>
      </div>

      <div class="emotion-evolution-section">
        <p class="emotion-evolution-label">依据</p>
        <p class="emotion-evolution-value emotion-evolution-basis">{{ props.user?.emotionBasis || '-' }}</p>
      </div>
    </div>
  </section>
</template>
