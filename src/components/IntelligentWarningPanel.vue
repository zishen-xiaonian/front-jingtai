<script setup>
import '../style.css'

const props = defineProps({
  user: {
    type: Object,
    default: null,
  },
})

const getWarningLevelClass = (level) => {
  const text = String(level || '')

  if (text.includes('中高风险')) {
    return 'orange'
  }

  if (text.includes('高风险')) {
    return 'danger'
  }

  if (text.includes('中风险')) {
    return 'warning'
  }

  if (text.includes('一般关注')) {
    return 'safe'
  }

  return ''
}
</script>

<template>
  <section class="module-block intelligent-warning-panel">
    <div class="block-head intelligent-warning-head">
      <h3>智能预警</h3>
    </div>

    <div class="intelligent-warning-content">
      <div class="intelligent-warning-section">
        <p class="intelligent-warning-label">预警等级信息</p>
        <p class="intelligent-warning-value" :class="getWarningLevelClass(props.user?.warningLevel)">
          {{ props.user?.warningLevel || '-' }}
        </p>
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
