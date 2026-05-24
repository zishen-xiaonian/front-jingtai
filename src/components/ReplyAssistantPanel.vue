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
  <section class="module-block reply-assistant-panel">
    <div class="block-head reply-assistant-head">
      <h3>智能回单助手</h3>
    </div>

    <div class="reply-assistant-template">
      <div class="reply-assistant-template-body">{{ props.user?.replyTemplate || '-' }}</div>
    </div>
  </section>
</template>
