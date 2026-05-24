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
  <section class="module-block intelligent-warning-panel">
    <div class="block-head">
      <h3>智能预警</h3>
    </div>

    <div class="intelligent-warning-content">
      <div class="intelligent-warning-section">
        <p class="intelligent-warning-label">预警等级信息</p>
        <p class="intelligent-warning-value">{{ props.user?.warningLevel || '-' }}</p>
      </div>

      <div class="intelligent-warning-section">
        <p class="intelligent-warning-label">预警依据</p>
        <div class="intelligent-warning-basis-list">
          <p v-for="item in props.user?.warningBasis || []" :key="item">{{ item }}</p>
          <p v-if="!props.user?.warningBasis?.length">-</p>
        </div>
      </div>
    </div>
  </section>
</template>
