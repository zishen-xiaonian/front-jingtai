<script setup>
import '../style.css'
import { computed, ref, watch } from 'vue'

const props = defineProps({
  countyWarningLights: {
    type: Array,
    required: true,
  },
  loading: {
    type: Boolean,
    default: false,
  },
  popupVisible: {
    type: Boolean,
    default: false,
  },
  popupCountyName: {
    type: String,
    default: '',
  },
  popupEvents: {
    type: Array,
    default: () => [],
  },
  popupLoading: {
    type: Boolean,
    default: false,
  },
  popupError: {
    type: String,
    default: '',
  },
})

const emit = defineEmits(['select-county', 'close-popup'])

const POPUP_PAGE_SIZE = 3

const popupCurrentPage = ref(1)

const popupTotalPages = computed(() => Math.max(1, Math.ceil(props.popupEvents.length / POPUP_PAGE_SIZE)))

const popupPageStartIndex = computed(() => (popupCurrentPage.value - 1) * POPUP_PAGE_SIZE)

const pagedPopupEvents = computed(() =>
  props.popupEvents.slice(popupPageStartIndex.value, popupPageStartIndex.value + POPUP_PAGE_SIZE),
)

const popupPageItems = computed(() => {
  const total = popupTotalPages.value
  const current = popupCurrentPage.value

  if (total <= 7) {
    return Array.from({ length: total }, (_, index) => index + 1)
  }

  const pages = new Set([1, total, current, current - 1, current + 1])

  if (current <= 3) {
    pages.add(2)
    pages.add(3)
    pages.add(4)
  }

  if (current >= total - 2) {
    pages.add(total - 3)
    pages.add(total - 2)
    pages.add(total - 1)
  }

  const sortedPages = Array.from(pages)
    .filter((page) => page >= 1 && page <= total)
    .sort((a, b) => a - b)

  return sortedPages.reduce((items, page, index) => {
    const previousPage = sortedPages[index - 1]

    if (previousPage && page - previousPage > 1) {
      items.push(`ellipsis-${previousPage}-${page}`)
    }

    items.push(page)
    return items
  }, [])
})

const setPopupPage = (page) => {
  if (typeof page !== 'number') {
    return
  }

  popupCurrentPage.value = Math.min(Math.max(page, 1), popupTotalPages.value)
}

watch(
  () => [props.popupVisible, props.popupCountyName],
  () => {
    popupCurrentPage.value = 1
  },
)

watch(popupTotalPages, (total) => {
  if (popupCurrentPage.value > total) {
    popupCurrentPage.value = total
  }
})
</script>

<template>
  <article class="module-block county-warning-lights-block">
    <div class="county-warning-lights-head">
      <h3>区县警示灯</h3>
      <p>唐山全区县停电状态</p>
    </div>

    <ul class="county-warning-lights-grid">
      <li v-for="item in props.countyWarningLights" :key="item.countyName" class="county-warning-light-item">
        <button
          type="button"
          class="county-warning-light-btn"
          :class="{ active: props.popupVisible && props.popupCountyName === item.countyName }"
          @click="emit('select-county', item)"
        >
          <span class="county-warning-light-dot" :class="item.hasOutage ? 'danger' : 'safe'"></span>
          <span class="county-warning-light-name">{{ item.countyName }}</span>
        </button>
      </li>
      <li v-if="props.loading && props.countyWarningLights.length === 0" class="county-warning-light-loading">
        <span class="county-warning-light-spinner" aria-hidden="true"></span>
        <span>数据加载中...</span>
      </li>
    </ul>

    <teleport to="body">
      <div v-if="props.popupVisible" class="county-warning-popup-mask">
        <div class="county-warning-popup-card">
          <div class="county-warning-popup-head">
            <h4>{{ props.popupCountyName || '供电单位' }}</h4>
            <button type="button" class="county-warning-popup-close" aria-label="关闭" @click="emit('close-popup')">×</button>
          </div>
          <div class="county-warning-popup-body">
            <div v-if="props.popupLoading" class="county-warning-popup-loading">
              <span class="county-warning-light-spinner" aria-hidden="true"></span>
              <span>停电事件加载中...</span>
            </div>
            <p v-else-if="props.popupError" class="county-warning-popup-empty">{{ props.popupError }}</p>
            <p v-else-if="props.popupEvents.length === 0" class="county-warning-popup-empty">当前暂无停电事件</p>
            <template v-else>
              <ul class="county-warning-event-list">
                <li
                  v-for="(event, index) in pagedPopupEvents"
                  :key="event?.id || event?.outageNumber || `${popupCurrentPage}-${index}`"
                  class="county-warning-event-item"
                >
                  <div class="county-warning-event-title-row">
                    <span class="county-warning-event-title">停电事件编号：{{ event.outageNumber || '-' }}</span>
                  </div>
                  <p>受影响用户总数：{{ event.affectedConsCnt ?? 0 }}户</p>
                  <p>受影响重要用户数：{{ event.importantUserCount ?? 0 }}户</p>
                  <p>受影响敏感用户数：{{ event.sensitiveUserCount ?? 0 }}户</p>
                </li>
              </ul>

              <div v-if="popupTotalPages > 1" class="county-warning-pagination" aria-label="停电事件分页">
                <template v-for="pageItem in popupPageItems" :key="pageItem">
                  <span
                    v-if="typeof pageItem !== 'number'"
                    class="county-warning-page-ellipsis"
                    aria-hidden="true"
                  >
                    ...
                  </span>
                  <button
                    v-else
                    type="button"
                    class="county-warning-page-btn"
                    :class="{ active: popupCurrentPage === pageItem }"
                    :aria-current="popupCurrentPage === pageItem ? 'page' : undefined"
                    @click="setPopupPage(pageItem)"
                  >
                    {{ pageItem }}
                  </button>
                </template>
              </div>
            </template>
          </div>
        </div>
      </div>
    </teleport>
  </article>
</template>

