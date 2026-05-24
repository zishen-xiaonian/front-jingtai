<script setup>
import '../style.css'
import { computed, ref, watch } from 'vue'
import { queryCountyEquipmentDetail } from '../api/outage'

const props = defineProps({
  rows: {
    type: Array,
    default: () => [],
  },
  selectedRegion: {
    type: String,
    default: '全部',
  },
  title: {
    type: String,
    default: '空间分布',
  },
  importantLabel: {
    type: String,
    default: '重点用户户数',
  },
  sensitiveLabel: {
    type: String,
    default: '敏感用户户数',
  },
  detailRows: {
    type: Array,
    default: () => [],
  },
  tableRows: {
    type: Array,
    default: () => [],
  },
  tableTotal: {
    type: Number,
    default: 0,
  },
  loading: {
    type: Boolean,
    default: false,
  },
  overviewLoading: {
    type: Boolean,
    default: false,
  },
  equipmentStatsLoading: {
    type: Boolean,
    default: false,
  },
})
const emit = defineEmits(['open-detail-page', 'go-detail-page', 'close-detail-page', 'select-top-device'])

const toCount = (value) => Math.max(0, Number(value) || 0)
const TREND_MIN_TRACK_PERCENT = 16

const normalizedRows = computed(() =>
  props.rows.map((item, index) => {
    const name = String(item?.name || item?.label || '').trim() || `数据${index + 1}`
    const importantCount = toCount(item?.importantCount ?? item?.important)
    const sensitiveCount = toCount(item?.sensitiveCount ?? item?.sensitive)
    return {
      name,
      importantCount,
      sensitiveCount,
      totalCount: importantCount + sensitiveCount,
    }
  }),
)

const maxTotal = computed(() => normalizedRows.value.reduce((max, item) => Math.max(max, item.totalCount), 0))

const displayRows = computed(() => {
  const base = maxTotal.value || 1
  return normalizedRows.value.map((item) => {
    const total = item.totalCount
    const rawTrackPercent = total > 0 ? (total / base) * 100 : 0
    return {
      ...item,
      trackPercent: total > 0 ? Math.max(rawTrackPercent, TREND_MIN_TRACK_PERCENT) : 0,
      importantPercent: total > 0 ? (item.importantCount / total) * 100 : 0,
      sensitivePercent: total > 0 ? (item.sensitiveCount / total) * 100 : 0,
    }
  })
})

const emptyText = computed(() => '')

const buildSegmentStyle = (percent, count) => {
  if (count <= 0 || percent <= 0) {
    return { width: '0%' }
  }

  return {
    width: `${percent}%`,
    minWidth: '28px',
  }
}

const detailPageVisible = ref(false)
const detailModalVisible = ref(false)
const selectedDetailRow = ref(null)
const detailModalLoading = ref(false)
const detailModalError = ref('')
const detailModalRequestId = ref(0)
const selectedTopDeviceId = ref('')
const DETAIL_PAGE_SIZE = 10
const DETAIL_PAGE_MAX_BUTTONS = 5
const detailCurrentPage = ref(1)
const detailJumpPageInput = ref('')

const normalizeEquipmentRows = (rows) =>
  rows.map((item, index) => {
    const equipmentId = String(item?.equipmentId || item?.deviceNo || '').trim() || '-'
    const equipmentType = String(item?.equipmentType || item?.equipment_type || item?.type || '').trim()
    const deviceNo = String(item?.deviceNo || '').trim() || '-'
    const deviceName = String(item?.deviceName || '').trim() || '-'
    const importantUserCount = toCount(item?.importantUserCount)
    const sensitiveUserCount = toCount(item?.sensitiveUserCount)
    const outageEventCount = toCount(item?.outageEventCount)
    const importantUserList = Array.isArray(item?.importantUserList) ? item.importantUserList : []
    const sensitiveUserList = Array.isArray(item?.sensitiveUserList) ? item.sensitiveUserList : []

    return {
      id: String(item?.key || `${deviceNo}-${deviceName}-${index}`),
      equipmentId,
      equipmentType,
      deviceNo,
      deviceName,
      importantUserCount,
      sensitiveUserCount,
      outageEventCount,
      totalUserCount: importantUserCount + sensitiveUserCount,
      importantUserList,
      sensitiveUserList,
    }
  })

const normalizedDetailRows = computed(() => normalizeEquipmentRows(props.detailRows))
const normalizedTableRows = computed(() => normalizeEquipmentRows(props.tableRows))

const top5RankedRows = computed(() => normalizedDetailRows.value.slice(0, 5))
const deviceImpactEmptyText = computed(() =>
  props.loading ? '数据加载中...' : '当前区域暂无设备影响用户数据。',
)
const selectedDeviceEmptyText = computed(() =>
  props.loading ? '数据加载中...' : '暂无设备数据。',
)

const detailTotalPages = computed(() => Math.max(Math.ceil(props.tableTotal / DETAIL_PAGE_SIZE), 1))

const pagedDetailRows = computed(() => normalizedTableRows.value)

const detailPageButtons = computed(() => {
  const total = detailTotalPages.value
  if (total <= DETAIL_PAGE_MAX_BUTTONS) {
    return Array.from({ length: total }, (_, index) => index + 1)
  }

  const half = Math.floor(DETAIL_PAGE_MAX_BUTTONS / 2)
  let start = detailCurrentPage.value - half
  let end = detailCurrentPage.value + half
  if (start < 1) {
    start = 1
    end = DETAIL_PAGE_MAX_BUTTONS
  }
  if (end > total) {
    end = total
    start = total - DETAIL_PAGE_MAX_BUTTONS + 1
  }

  const pages = []
  for (let page = start; page <= end; page += 1) {
    pages.push(page)
  }
  return pages
})

const topDeviceMaxImpact = computed(() =>
  top5RankedRows.value.reduce((max, item) => Math.max(max, item.importantUserCount), 0),
)

const selectedTopDevice = computed(() => {
  if (!selectedTopDeviceId.value) {
    return top5RankedRows.value[0] || null
  }

  return top5RankedRows.value.find((item) => item.id === selectedTopDeviceId.value) || top5RankedRows.value[0] || null
})

const summaryStats = computed(() => {
  const rows = normalizedDetailRows.value
  const affectedDeviceCount = rows.length
  const importantUserTotal = rows.reduce((sum, item) => sum + item.importantUserCount, 0)
  const sensitiveUserTotal = rows.reduce((sum, item) => sum + item.sensitiveUserCount, 0)
  const highRiskDeviceCount = rows.filter((item) => item.totalUserCount >= 20).length
  const highRiskRatio = affectedDeviceCount > 0 ? (highRiskDeviceCount / affectedDeviceCount) * 100 : 0

  return {
    affectedDeviceCount,
    importantUserTotal,
    sensitiveUserTotal,
    highRiskRatio,
  }
})

watch(
  top5RankedRows,
  (rows) => {
    if (rows.length === 0) {
      selectedTopDeviceId.value = ''
      return
    }

    const exists = rows.some((item) => item.id === selectedTopDeviceId.value)
    if (!exists) {
      selectedTopDeviceId.value = rows[0].id
    }
  },
  { immediate: true },
)

watch(
  detailTotalPages,
  (total) => {
    if (detailCurrentPage.value > total) {
      detailCurrentPage.value = total
    }
  },
  { immediate: true },
)

const openDetailPage = () => {
  detailPageVisible.value = true
  detailCurrentPage.value = 1
  detailJumpPageInput.value = ''
  emit('open-detail-page')
}

const closeDetailPage = () => {
  detailPageVisible.value = false
  detailModalVisible.value = false
  detailModalLoading.value = false
  detailModalError.value = ''
  selectedDetailRow.value = null
  detailModalRequestId.value += 1
  detailCurrentPage.value = 1
  detailJumpPageInput.value = ''
  emit('close-detail-page')
}

const buildUserDisplayText = (record) => {
  const consNo = String(record?.consNo || record?.cons_no || record?.userNo || record?.user_id || '').trim() || '-'
  const consName = String(record?.consName || record?.cons_name || record?.userName || record?.name || '').trim() || '-'
  const countyName = String(record?.countyName || record?.county_name || '').trim() || '-'
  const consAddr = String(record?.consAddr || record?.cons_addr || record?.consAddress || record?.address || '').trim() || '-'
  return `${consNo} / ${consName} / ${countyName} / ${consAddr}`
}

const mapDetailUsers = (users) =>
  (Array.isArray(users) ? users : []).map((item) => buildUserDisplayText(item))

const openDetailModal = async (item) => {
  const equipmentId = String(item?.equipmentId || item?.deviceNo || '').trim()
  selectedDetailRow.value = {
    ...(item || {}),
    equipmentId: equipmentId || '-',
    deviceNo: String(item?.deviceNo || equipmentId || '').trim() || '-',
    deviceName: String(item?.deviceName || '').trim() || '-',
    importantUserCount: toCount(item?.importantUserCount),
    sensitiveUserCount: toCount(item?.sensitiveUserCount),
    importantUserList: Array.isArray(item?.importantUserList) ? item.importantUserList : [],
    sensitiveUserList: Array.isArray(item?.sensitiveUserList) ? item.sensitiveUserList : [],
  }
  emit('select-top-device', selectedDetailRow.value)
  detailModalVisible.value = true
  detailModalLoading.value = true
  detailModalError.value = ''

  if (!equipmentId) {
    detailModalLoading.value = false
    detailModalError.value = 'Missing equipment ID. Unable to load details.'
    return
  }

  const requestId = detailModalRequestId.value + 1
  detailModalRequestId.value = requestId

  try {
    const response = await queryCountyEquipmentDetail({ equipmentId })
    if (requestId !== detailModalRequestId.value) {
      return
    }

    const data = response?.data || {}
    const importantUserList = mapDetailUsers(data?.keyUsers)
    const sensitiveUserList = mapDetailUsers(data?.sensitiveUsers)

    selectedDetailRow.value = {
      ...(selectedDetailRow.value || {}),
      equipmentId: String(data?.equipmentId || equipmentId).trim() || '-',
      equipmentType: String(data?.equipmentType || selectedDetailRow.value?.equipmentType || '').trim(),
      deviceNo: String(data?.equipmentId || selectedDetailRow.value?.deviceNo || equipmentId).trim() || '-',
      deviceName: String(data?.equipmentName || selectedDetailRow.value?.deviceName || '').trim() || '-',
      importantUserCount: toCount(data?.keyUserCount ?? importantUserList.length),
      sensitiveUserCount: toCount(data?.sensitiveUserCount ?? sensitiveUserList.length),
      totalUserCount: toCount(data?.keyUserCount ?? importantUserList.length) + toCount(data?.sensitiveUserCount ?? sensitiveUserList.length),
      importantUserList,
      sensitiveUserList,
    }
    emit('select-top-device', selectedDetailRow.value)
  } catch {
    if (requestId !== detailModalRequestId.value) {
      return
    }
    detailModalError.value = 'Failed to load details. Please try again.'
  } finally {
    if (requestId === detailModalRequestId.value) {
      detailModalLoading.value = false
    }
  }
}

const closeDetailModal = () => {
  detailModalVisible.value = false
  detailModalLoading.value = false
  detailModalError.value = ''
  selectedDetailRow.value = null
  detailModalRequestId.value += 1
}

const selectTopDevice = (item) => {
  selectedTopDeviceId.value = item?.id || ''
  emit('select-top-device', item || null)
}

const goDetailPage = (page) => {
  if (!Number.isFinite(page)) {
    return
  }
  const targetPage = Math.max(1, Math.min(detailTotalPages.value, Math.round(page)))
  detailCurrentPage.value = targetPage
  detailJumpPageInput.value = ''
  emit('go-detail-page', targetPage)
}

const jumpToDetailPage = () => {
  const input = String(detailJumpPageInput.value ?? '').trim()
  if (!input) {
    return
  }
  const parsed = Number(input)
  if (!Number.isFinite(parsed)) {
    return
  }
  goDetailPage(parsed)
}

const buildTopBarStyle = (totalCount) => {
  if (totalCount <= 0 || topDeviceMaxImpact.value <= 0) {
    return { width: '0%' }
  }

  return {
    width: `${(totalCount / topDeviceMaxImpact.value) * 100}%`,
  }
}

const formatPercent = (value) => `${value.toFixed(1)}%`
</script>

<template>
  <section
    class="key-user-time-trend-card module-clickable space-distribution-card"
    role="button"
    tabindex="0"
    @click="!detailPageVisible && openDetailPage()"
    @keydown.enter.prevent="!detailPageVisible && openDetailPage()"
    @keydown.space.prevent="!detailPageVisible && openDetailPage()"
  >
    <header class="trend-head">
      <h4>{{ props.title }}</h4>
      <div class="trend-legend" aria-label="图例">
        <span class="trend-legend-item important">
          <i aria-hidden="true"></i>
          {{ props.importantLabel }}
        </span>
        <span class="trend-legend-item sensitive">
          <i aria-hidden="true"></i>
          {{ props.sensitiveLabel }}
        </span>
      </div>
    </header>

    <div class="trend-chart-wrap space-trend-chart-wrap">
      <div v-if="displayRows.length > 0" class="trend-rows">
        <div v-for="row in displayRows" :key="row.name" class="trend-row">
          <div class="trend-row-name" :title="row.name">{{ row.name }}</div>

          <div class="trend-row-bars">
            <div class="trend-row-track">
              <div class="trend-row-stack" :style="{ width: `${row.trackPercent}%` }">
                <div class="trend-row-segment important" :style="buildSegmentStyle(row.importantPercent, row.importantCount)">
                  <span v-if="row.importantCount > 0" class="trend-row-value">{{ row.importantCount }}</span>
                </div>

                <div class="trend-row-segment sensitive" :style="buildSegmentStyle(row.sensitivePercent, row.sensitiveCount)">
                  <span v-if="row.sensitiveCount > 0" class="trend-row-value">{{ row.sensitiveCount }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else-if="props.overviewLoading" class="trend-empty space-overview-loading">
        <span class="space-loading-spinner" aria-hidden="true"></span>
        <span>数据加载中...</span>
      </div>

      <div v-else class="trend-empty">{{ emptyText }}</div>
    </div>

    <section v-if="detailPageVisible" class="card key-user-detail-layer" @click.stop>
      <header class="key-user-detail-layer-head">
        <h3>空间分布</h3>
        <button type="button" class="user-detail-close" @click.stop="closeDetailPage">×</button>
      </header>

      <div class="space-detail-layout">
        <section class="space-summary-panel">
          <h4>设备影响摘要</h4>
          <div class="space-summary-grid">
            <article class="space-summary-item">
              <p>影响设备数</p>
              <strong v-if="props.equipmentStatsLoading" class="space-summary-loading-text">数据加载中...</strong>
              <strong v-else>{{ summaryStats.affectedDeviceCount }}</strong>
            </article>
            <article class="space-summary-item">
              <p>敏感用户总数</p>
              <strong v-if="props.equipmentStatsLoading" class="space-summary-loading-text">数据加载中...</strong>
              <strong v-else>{{ summaryStats.sensitiveUserTotal }}</strong>
            </article>
            <article class="space-summary-item">
              <p>重点用户总数</p>
              <strong v-if="props.equipmentStatsLoading" class="space-summary-loading-text">数据加载中...</strong>
              <strong v-else>{{ summaryStats.importantUserTotal }}</strong>
            </article>
            <article class="space-summary-item">
              <p>高风险设备占比</p>
              <strong v-if="props.equipmentStatsLoading" class="space-summary-loading-text">数据加载中...</strong>
              <strong v-else>{{ formatPercent(summaryStats.highRiskRatio) }}</strong>
              <small>重点+敏感 >= 20</small>
            </article>
          </div>
        </section>

        <section class="space-rank-detail-row">
          <article class="space-rank-card">
            <h4>Top5 影响排行</h4>
            <ul v-if="top5RankedRows.length > 0" class="space-rank-list">
              <li v-for="(item, index) in top5RankedRows" :key="item.id">
                <button
                  type="button"
                  class="space-rank-item"
                  :class="{ active: selectedTopDevice && selectedTopDevice.id === item.id }"
                  @click.stop="selectTopDevice(item)"
                >
                  <div class="space-rank-item-head">
                    <span class="space-rank-no">{{ index + 1 }}</span>
                    <span class="space-rank-name" :title="item.deviceName">{{ item.deviceName }}</span>
                    <span class="space-rank-count">{{ item.importantUserCount }}</span>
                  </div>
                  <span class="space-rank-bar-bg">
                    <span class="space-rank-bar-fill" :style="buildTopBarStyle(item.importantUserCount)"></span>
                  </span>
                </button>
              </li>
            </ul>
            <p v-else class="empty-tip">{{ deviceImpactEmptyText }}</p>
          </article>

          <article class="space-selected-card">
            <h4>选中设备详情</h4>
            <div v-if="selectedTopDevice" class="space-selected-fields">
              <p><span>设备编号：</span>{{ selectedTopDevice.deviceNo }}</p>
              <p><span>设备名称：</span>{{ selectedTopDevice.deviceName }}</p>
              <p><span>重点用户数：</span>{{ selectedTopDevice.importantUserCount }}</p>
              <p><span>敏感用户数：</span>{{ selectedTopDevice.sensitiveUserCount }}</p>
              <p><span>停电事件数：</span>{{ selectedTopDevice.outageEventCount }}</p>
              <p><span>影响用户总数：</span>{{ selectedTopDevice.importantUserCount }}</p>
            </div>
            <p v-else class="empty-tip">{{ selectedDeviceEmptyText }}</p>
          </article>
        </section>

        <section class="space-detail-table-row key-user-table-module user-detail-table-module">
          <div class="space-detail-grid-wrap key-user-grid-wrap user-detail-grid-wrap">
            <ul class="key-user-grid-body user-detail-grid-body">
              <li class="key-user-grid user-detail-grid user-detail-grid-head">
                <span>设备编号</span>
                <span>设备名称</span>
                <span>重点用户数</span>
                <span>敏感用户数</span>
                <span>详情</span>
              </li>
              <li
                v-for="item in props.loading ? [] : pagedDetailRows"
                :key="item.id"
                class="key-user-grid user-detail-grid user-detail-grid-row"
              >
                <span class="user-detail-cell" :title="item.deviceNo">{{ item.deviceNo }}</span>
                <span class="user-detail-cell" :title="item.deviceName">{{ item.deviceName }}</span>
                <span class="user-detail-cell">{{ item.importantUserCount }}</span>
                <span class="user-detail-cell">{{ item.sensitiveUserCount }}</span>
                <button type="button" class="detail-btn" @click.stop="openDetailModal(item)">详情</button>
              </li>
            </ul>

            <p v-if="props.loading || normalizedTableRows.length === 0" class="empty-tip">{{ deviceImpactEmptyText }}</p>
          </div>

          <footer class="user-detail-pagination" v-if="props.tableTotal > 0">
            <button
              v-for="page in detailPageButtons"
              :key="`space-detail-page-${page}`"
              type="button"
              class="page-btn"
              :class="{ active: page === detailCurrentPage }"
              @click="goDetailPage(page)"
            >
              {{ page }}
            </button>

            <div class="user-detail-page-jump">
              <input
                v-model="detailJumpPageInput"
                type="number"
                min="1"
                :max="detailTotalPages"
                class="user-detail-page-input"
                placeholder="&#39029;&#30721;"
                @keyup.enter="jumpToDetailPage"
              />
              <button type="button" class="user-detail-page-jump-btn" @click="jumpToDetailPage">&#36339;&#36716;</button>
            </div>
          </footer>
        </section>
      </div>

      <div v-if="detailModalVisible && selectedDetailRow" class="user-detail-modal-mask" @click.self="closeDetailModal">
        <article class="user-detail-modal key-user-detail-modal">
          <button type="button" class="user-detail-modal-close" @click="closeDetailModal">×</button>
          <h4>设备影响用户详情</h4>
          <div
            class="user-detail-modal-content"
            :class="{ 'sensitive-list-empty': selectedDetailRow.sensitiveUserList.length === 0 }"
          >
            <p><span>设备编号：</span>{{ selectedDetailRow.deviceNo }}</p>
            <p><span>设备名称：</span>{{ selectedDetailRow.deviceName }}</p>
            <p><span>重点用户数：</span>{{ selectedDetailRow.importantUserCount }}</p>
            <p><span>敏感用户数：</span>{{ selectedDetailRow.sensitiveUserCount }}</p>

            <div class="space-user-list-card important-list-card">
              <p class="space-user-list-title">重点用户清单：</p>
              <ul v-if="selectedDetailRow.importantUserList.length > 0" class="space-user-list important-user-list">
                <li v-for="item in selectedDetailRow.importantUserList" :key="`important-${item}`">{{ item }}</li>
              </ul>
              <p v-else class="space-user-list-empty">无</p>
            </div>

            <div
              class="space-user-list-card sensitive-list-card"
              :class="{ 'is-empty': selectedDetailRow.sensitiveUserList.length === 0 }"
            >
              <p class="space-user-list-title">敏感用户清单：</p>
              <ul v-if="selectedDetailRow.sensitiveUserList.length > 0" class="space-user-list">
                <li v-for="item in selectedDetailRow.sensitiveUserList" :key="`sensitive-${item}`">{{ item }}</li>
              </ul>
              <p v-else class="space-user-list-empty">无</p>
            </div>
          </div>
        </article>
      </div>
    </section>
  </section>
</template>

