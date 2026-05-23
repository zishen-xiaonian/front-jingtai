<script setup>
import '../style.css'
import { computed, ref } from 'vue'

const props = defineProps({
  selectedFaultCounty: {
    type: String,
    default: '',
  },
  countyRegionOptions: {
    type: Array,
    required: true,
  },
  faultLocationSummary: {
    type: Object,
    required: true,
  },
  filteredFaultOutageEventsLength: {
    type: Number,
    default: 0,
  },
  faultLocationLoading: {
    type: Boolean,
    default: false,
  },
  showOutageDetailPage: {
    type: Boolean,
    default: false,
  },
  outageNatureOverview: {
    type: Object,
    required: true,
  },
  outageEventsSummaryLoading: {
    type: Boolean,
    default: false,
  },
  outageRestoreOverview: {
    type: Object,
    required: true,
  },
  outageDetailSearchInput: {
    type: String,
    default: '',
  },
  outageDetailSelectedNature: {
    type: String,
    default: '',
  },
  pagedOutageDetailRows: {
    type: Array,
    required: true,
  },
  filteredOutageDetailRowsLength: {
    type: Number,
    default: 0,
  },
  outageDetailPageButtons: {
    type: Array,
    required: true,
  },
  outageDetailCurrentPage: {
    type: Number,
    default: 1,
  },
  outageDetailJumpPageInput: {
    type: [String, Number],
    default: '',
  },
  outageDetailTotalPages: {
    type: Number,
    default: 1,
  },
  outageDetailLoading: {
    type: Boolean,
    default: false,
  },
  outageDetailModalVisible: {
    type: Boolean,
    default: false,
  },
  outageDetailDimension: {
    type: String,
    default: 'feeder',
  },
  selectedOutageDetail: {
    type: Object,
    default: null,
  },
  outageDetailGridBodyRefSetter: {
    type: Function,
    default: null,
  },
  outageDetailPaginationRefSetter: {
    type: Function,
    default: null,
  },
  outageDetailPageJumpRefSetter: {
    type: Function,
    default: null,
  },
})

const emit = defineEmits([
  'update:selected-fault-county',
  'open-outage-detail',
  'fault-mode-change',
  'close-outage-detail',
  'update:outage-detail-search-input',
  'apply-outage-detail-search',
  'update:outage-detail-selected-nature',
  'open-outage-detail-modal',
  'go-outage-detail-page',
  'update:outage-detail-jump-page-input',
  'jump-to-outage-detail-page',
  'close-outage-detail-modal',
])

const onCountyChange = (event) => {
  emit('update:selected-fault-county', event?.target?.value || '')
}

const updateOutageDetailSearchInput = (event) => {
  emit('update:outage-detail-search-input', event?.target?.value || '')
}

const updateOutageDetailSelectedNature = (event) => {
  emit('update:outage-detail-selected-nature', event?.target?.value || '')
}

const updateOutageDetailJumpPageInput = (event) => {
  emit('update:outage-detail-jump-page-input', event?.target?.value || '')
}

const modeOptions = [
  { key: 'feeder', label: '线路' },
  { key: 'substation', label: '变电站' },
]
const legendItems = [
  { key: 'danger', label: '影响用户数：大于5000' },
  { key: 'warning', label: '影响用户数：1000-5000' },
  { key: 'safe', label: '影响用户数：小于1000' },
]

const activeFaultMode = ref('feeder')

const activeModeSummary = computed(() => {
  const fallback = {
    key: activeFaultMode.value,
    label: modeOptions.find((item) => item.key === activeFaultMode.value)?.label || '线路',
    total: 0,
    matchedEvents: 0,
    bars: [
      { key: 'danger', colorLabel: '红色', count: 0 },
      { key: 'warning', colorLabel: '黄色', count: 0 },
      { key: 'safe', colorLabel: '绿色', count: 0 },
    ],
  }
  return props.faultLocationSummary?.modes?.[activeFaultMode.value] || fallback
})

const activeModeBars = computed(() => {
  const bars = Array.isArray(activeModeSummary.value?.bars) ? activeModeSummary.value.bars : []
  const maxCount = Math.max(...bars.map((item) => Number(item?.count || 0)), 1)
  return bars.map((item) => {
    const count = Number(item?.count || 0)
    const basePercent = maxCount > 0 ? (count / maxCount) * 100 : 0
    const heightPercent = count > 0 ? Math.max(basePercent, 18) : 0
    return {
      ...item,
      count,
      heightPercent,
    }
  })
})

const setFaultMode = (modeKey) => {
  if (activeFaultMode.value === modeKey) {
    return
  }
  activeFaultMode.value = modeKey
  emit('fault-mode-change', modeKey)
}
</script>

<template>
  <article class="module-block module-clickable" @click="emit('open-outage-detail', activeFaultMode)">
    <div class="fault-location-top">
      <div class="block-head fault-location-head">
        <h3>电力故障定位</h3>
      </div>
      <ul class="fault-legend-list" @click.stop>
        <li v-for="item in legendItems" :key="item.key" class="fault-legend-item">
          <span class="fault-legend-dot" :class="item.key"></span>
          <span>{{ item.label }}</span>
        </li>
      </ul>
    </div>

    <div class="fault-mode-switch" @click.stop>
      <button
        v-for="mode in modeOptions"
        :key="mode.key"
        type="button"
        class="fault-mode-btn"
        :class="{ active: mode.key === activeFaultMode }"
        @click.stop="setFaultMode(mode.key)"
      >
        {{ mode.label }}
      </button>
    </div>

    <div class="fault-bar-chart-wrap">
      <span class="fault-axis-label">数量</span>
      <div class="fault-bar-chart">
        <div v-for="bar in activeModeBars" :key="`${activeModeSummary.key}-${bar.key}`" class="fault-bar-item">
          <div class="fault-bar-zone">
            <span class="fault-bar-count" :class="{ zero: bar.count === 0 }">{{ bar.count }}</span>
            <span v-if="bar.count > 0" class="fault-bar" :class="bar.key" :style="{ height: `${bar.heightPercent}%` }"></span>
          </div>
        </div>
      </div>
      <div v-if="props.faultLocationLoading" class="fault-location-loading">
        <span class="fault-location-spinner" aria-hidden="true"></span>
        <span>数据加载中...</span>
      </div>
    </div>

    <p class="event-meta">
      当前{{ activeModeSummary.label }}数量：{{ activeModeSummary.total }}，当前区域：{{ props.selectedFaultCounty || '-' }}，停电事件 {{ activeModeSummary.matchedEvents ?? props.filteredFaultOutageEventsLength }} 条
    </p>
  </article>

  <section v-if="props.showOutageDetailPage" class="card outage-detail-layer">
    <header class="outage-detail-layer-head">
      <h3>停电事件详情（{{ props.selectedFaultCounty || '-' }}）</h3>
      <button type="button" class="outage-detail-close" @click="emit('close-outage-detail')">×</button>
    </header>

    <section class="outage-nature-overview-module">
      <article class="outage-nature-pie-card">
        <div class="outage-nature-pie-head">
          <h4>停电性质占比</h4>
          <div class="outage-nature-pie-metas">
            <span class="outage-nature-pie-meta planned">
              计划停电 {{ props.outageNatureOverview.planned.count }}条 / {{ props.outageNatureOverview.planned.rateText }}
            </span>
            <span class="outage-nature-pie-meta fault">
              故障停电 {{ props.outageNatureOverview.fault.count }}条 / {{ props.outageNatureOverview.fault.rateText }}
            </span>
            <span class="outage-nature-pie-meta other">
              其他 {{ props.outageNatureOverview.other.count }}条 / {{ props.outageNatureOverview.other.rateText }}
            </span>
          </div>
        </div>

        <div class="outage-nature-pie-wrap">
          <div class="outage-nature-pie" :style="{ background: props.outageNatureOverview.pieBackground }">
            <div class="outage-nature-pie-center">
              <template v-if="props.outageEventsSummaryLoading">
                <strong class="outage-nature-pie-loading-text">数据加载中...</strong>
              </template>
              <template v-else>
                <strong>{{ props.outageNatureOverview.total }}</strong>
                <span>总停电事件</span>
              </template>
            </div>
          </div>
        </div>

        <p class="outage-nature-card-note"></p>
      </article>

      <article class="outage-restore-card">
        <div class="outage-restore-head">
          <h4>复电情况</h4>
          <span class="outage-restore-meta">
            已复电 {{ props.outageRestoreOverview.restored.count }}条 / {{ props.outageRestoreOverview.restored.rateText }}
          </span>
        </div>

        <div class="outage-restore-progress">
          <div class="outage-restore-progress-track">
            <span class="outage-restore-progress-fill" :style="{ width: props.outageRestoreOverview.restored.rateText }"></span>
          </div>
          <div class="outage-restore-progress-foot">
            <span>待复电 {{ props.outageRestoreOverview.unrestored.count }}条</span>
            <span>总事件 {{ props.outageRestoreOverview.total }}条</span>
          </div>
        </div>
      </article>
    </section>

    <section class="outage-detail-table-module">
      <div class="outage-detail-query-bar">
        <input
          :value="props.outageDetailSearchInput"
          type="text"
          class="outage-detail-query-input"
          placeholder="请输入事件ID"
          @input="updateOutageDetailSearchInput"
          @keyup.enter="emit('apply-outage-detail-search')"
        />
        <button type="button" class="outage-detail-query-btn" @click="emit('apply-outage-detail-search')">查询</button>
        <select :value="props.outageDetailSelectedNature" class="outage-detail-nature-select" @change="updateOutageDetailSelectedNature">
          <option value="">全部</option>
          <option value="计划停电">计划停电</option>
          <option value="故障停电">故障停电</option>
          <option value="其他">其他</option>
        </select>
      </div>

      <div class="outage-detail-grid-wrap">
        <ul :ref="props.outageDetailGridBodyRefSetter" class="outage-detail-grid-body">
          <li class="outage-detail-grid outage-detail-grid-head">
            <span>事件ID</span>
            <span>区县单位</span>
            <span>影响户数</span>
            <span>停电性质</span>
            <span>详情</span>
          </li>
          <li
            v-for="item in props.outageDetailLoading ? [] : props.pagedOutageDetailRows"
            :key="item.id"
            class="outage-detail-grid outage-detail-grid-row"
          >
            <span class="outage-detail-cell" :title="item.outageNumber">{{ item.outageNumber }}</span>
            <span class="outage-detail-cell" :title="item.countyName">{{ item.countyName }}</span>
            <span class="outage-detail-cell" :title="String(item.affectedConsCnt)">{{ item.affectedConsCnt }}</span>
            <span class="outage-detail-cell" :title="item.outageNature">{{ item.outageNature }}</span>
            <button type="button" class="detail-btn" @click="emit('open-outage-detail-modal', item)">详情</button>
          </li>
        </ul>

        <p v-if="props.outageDetailLoading || props.pagedOutageDetailRows.length === 0" class="empty-tip">
          {{ props.outageDetailLoading ? '数据加载中...' : '当前区域暂无停电事件详情。' }}
        </p>
      </div>

      <footer :ref="props.outageDetailPaginationRefSetter" class="user-detail-pagination" v-if="props.filteredOutageDetailRowsLength > 0">
        <button
          v-for="page in props.outageDetailPageButtons"
          :key="`outage-page-${page}`"
          type="button"
          class="page-btn"
          :class="{ active: page === props.outageDetailCurrentPage }"
          @click="emit('go-outage-detail-page', page)"
        >
          {{ page }}
        </button>

        <div :ref="props.outageDetailPageJumpRefSetter" class="outage-detail-page-jump">
          <input
            :value="props.outageDetailJumpPageInput"
            type="number"
            min="1"
            :max="props.outageDetailTotalPages"
            class="outage-detail-page-input"
            placeholder="页码"
            @input="updateOutageDetailJumpPageInput"
            @keyup.enter="emit('jump-to-outage-detail-page')"
          />
          <button type="button" class="outage-detail-page-jump-btn" @click="emit('jump-to-outage-detail-page')">跳转</button>
        </div>
      </footer>
    </section>

    <div
      v-if="props.outageDetailModalVisible && props.selectedOutageDetail"
      class="user-detail-modal-mask"
      @click.self="emit('close-outage-detail-modal')"
    >
      <article class="user-detail-modal outage-detail-modal">
        <button type="button" class="user-detail-modal-close" @click="emit('close-outage-detail-modal')">×</button>
        <h4>停电事件详情</h4>
        <div class="user-detail-modal-content">
          <p><span>停电开始时间：</span>{{ props.selectedOutageDetail.beginTime }}</p>
          <p><span>状态：</span>{{ props.selectedOutageDetail.status }}</p>
          <p><span>停电结束时间：</span>{{ props.selectedOutageDetail.endTime }}</p>
          <p><span>区县单位：</span>{{ props.selectedOutageDetail.countyName }}</p>
          <p><span>影响户数：</span>{{ props.selectedOutageDetail.affectedConsCnt }}</p>
          <p v-if="props.outageDetailDimension !== 'substation'"><span>停电线路：</span>{{ props.selectedOutageDetail.rdtFeederName }}</p>
          <p v-if="props.outageDetailDimension !== 'feeder'"><span>所属变电站：</span>{{ props.selectedOutageDetail.rdtSubsName }}</p>
        </div>
      </article>
    </div>
  </section>
</template>

