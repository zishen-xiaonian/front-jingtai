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
  <section class="module-block user-personal-info-panel">
    <div class="block-head user-personal-info-head">
      <h3>用户信息</h3>
    </div>

    <div class="user-personal-info-list">
      <div class="user-personal-info-row">
        <span class="user-personal-info-label">用户名</span>
        <span class="user-personal-info-value">{{ props.user?.consName || '-' }}</span>
      </div>
      <div class="user-personal-info-row">
        <span class="user-personal-info-label">用户编号</span>
        <span class="user-personal-info-value">{{ props.user?.consNo || '-' }}</span>
      </div>
      <div class="user-personal-info-row">
        <span class="user-personal-info-label">联系电话</span>
        <span class="user-personal-info-value">{{ props.user?.phone || '-' }}</span>
      </div>
      <div class="user-personal-info-row">
        <span class="user-personal-info-label">用户地址</span>
        <span class="user-personal-info-value">{{ props.user?.address || '-' }}</span>
      </div>
    </div>
  </section>
</template>
