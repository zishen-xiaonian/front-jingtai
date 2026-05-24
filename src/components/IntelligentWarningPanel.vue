<script setup>
import '../style.css'
</script>

<template>
  <section class="module-block intelligent-warning-panel">
    <div class="block-head">
      <h3>智能预警</h3>
    </div>

    <div class="intelligent-warning-content">
      <div class="intelligent-warning-section">
        <p class="intelligent-warning-label">预警等级信息</p>
        <p class="intelligent-warning-value"></p>
      </div>

      <div class="intelligent-warning-section">
        <p class="intelligent-warning-label">预警依据</p>
        <div class="intelligent-warning-basis-list">
          <p>该用户历史工单聚类</p>
          <p>该用户情绪演变过程</p>
        </div>
      </div>
    </div>
  </section>
</template>
