<script lang="ts" setup>
import {
  computed,
  inject,
  onBeforeUnmount,
  onMounted,
  PropType,
  reactive,
  ref,
  shallowRef,
  ShallowRef,
  toRaw,
  toRefs
} from 'vue'
import { getData } from '@/api/chart'
import chartViewManager from '@/views/chart/components/js/panel'
import { dvMainStoreWithOut } from '@/store/modules/data-visualization/dvMain'
import ViewTrackBar from '@/components/visualization/ViewTrackBar.vue'
import { storeToRefs } from 'pinia'
import { S2ChartView } from '@/views/chart/components/js/panel/types/impl/s2'
import { ElPagination } from 'element-plus-secondary'
import ChartError from '@/views/chart/components/views/components/ChartError.vue'
import { defaultsDeep, cloneDeep } from 'lodash-es'
import { BASE_VIEW_CONFIG } from '../../editor/util/chart'

const dvMainStore = dvMainStoreWithOut()
const { nowPanelTrackInfo, nowPanelJumpInfo } = storeToRefs(dvMainStore)

const props = defineProps({
  view: {
    type: Object as PropType<ChartObj>,
    default() {
      return {
        propValue: null
      }
    }
  },
  showPosition: {
    type: String,
    required: false,
    default: 'canvas'
  }
})

const emit = defineEmits(['onChartClick', 'onDrillFilters', 'onJumpClick'])

const { view, showPosition } = toRefs(props)

const isError = ref(false)
const errMsg = ref('')
const chartExtRequest = inject('chartExtRequest') as ShallowRef<object>

const state = reactive({
  trackBarStyle: {
    position: 'absolute',
    left: '50px',
    top: '50px'
  },
  linkageActiveParam: null,
  pointParam: null,
  myChart: null,
  loading: false,
  data: { fields: [] }, // 图表数据
  pageInfo: {
    total: 0,
    pageSize: 20,
    currentPage: 1
  },
  totalItems: 0,
  showPage: false
})
// 图表数据不用全响应式
let chartData = shallowRef<Partial<Chart['data']>>({
  fields: []
})

const containerId = 'container-' + showPosition.value + '-' + view.value.id
const viewTrack = ref(null)

const calcData = (view: Chart, callback, resetPageInfo = true) => {
  if (view.tableId) {
    isError.value = false
    const v = JSON.parse(JSON.stringify(view))
    getData(v)
      .then(res => {
        if (res.code && res.code !== 0) {
          isError.value = true
          errMsg.value = res.msg
        } else {
          chartData.value = res?.data as Partial<Chart['data']>
          state.totalItems = res?.totalItems
          dvMainStore.setViewDataDetails(view.id, chartData.value)
          emit('onDrillFilters', res?.drillFilters)
          renderChart(res as unknown as Chart, resetPageInfo)
        }
        callback?.()
      })
      .catch(() => {
        callback?.()
      })
  } else {
    callback?.()
  }
}
// 图表对象不用响应式
let myChart = null
const renderChartFromDialog = (viewInfo: Chart, chartDataInfo) => {
  chartData.value = chartDataInfo
  renderChart(viewInfo, false)
}
const renderChart = (view: Chart, resetPageInfo: boolean) => {
  if (!view) {
    return
  }
  // view 为引用对象 需要存库 view.data 直接赋值会导致保存不必要的数据
  const chart = {
    ...defaultsDeep(view, cloneDeep(BASE_VIEW_CONFIG)),
    data: chartData.value
  } as ChartObj
  setupPage(chart, resetPageInfo)
  myChart?.destroy()
  const chartView = chartViewManager.getChartView(view.render, view.type) as S2ChartView<any>
  myChart = chartView.drawChart({
    container: containerId,
    chart: toRaw(chart),
    chartObj: myChart,
    pageInfo: state.pageInfo,
    action
  })
  myChart?.render()
  initScroll()
}

const pageColor = computed(() => {
  const text = view.value?.customStyle?.text
  return text.color ?? 'white'
})
const setupPage = (chart: ChartObj, resetPageInfo?: boolean) => {
  const customAttr = chart.customAttr
  if (chart.type !== 'table-info' || customAttr.basicStyle.tablePageMode !== 'page') {
    state.showPage = false
    return
  }
  const pageInfo = state.pageInfo
  pageInfo.pageSize = customAttr.basicStyle.tablePageSize ?? 20
  if (state.totalItems > state.pageInfo.pageSize) {
    pageInfo.total = state.totalItems
    state.showPage = true
  } else {
    state.showPage = false
  }
  if (resetPageInfo) {
    state.pageInfo.currentPage = 1
  }
}

let scrollTimer
let scrollTop = 0
const initScroll = () => {
  clearInterval(scrollTimer)
  // 首先回到最顶部，然后计算行高*行数作为top，最后判断：如果top<数据量*行高，继续滚动，否则回到顶部
  const customAttr = view.value.customAttr
  const senior = view.value.senior
  scrollTop = 0
  if (
    senior?.scrollCfg?.open &&
    (view.value.type === 'table-normal' || (view.value.type === 'table-info' && !state.showPage))
  ) {
    const rowHeight = customAttr.tableCell.tableItemHeight
    const headerHeight = customAttr.tableHeader.tableTitleHeight
    const dom = document.getElementById(containerId)
    if (dom.offsetHeight > rowHeight * chartData.value.tableRow.length + headerHeight) {
      return
    }
    scrollTimer = setInterval(() => {
      const offsetHeight = dom.offsetHeight
      const top = rowHeight * senior.scrollCfg.row
      if (offsetHeight - headerHeight + scrollTop < rowHeight * chartData.value.tableRow.length) {
        scrollTop += top
      } else {
        scrollTop = 0
      }
      myChart.store.set('scrollY', scrollTop)
      if (!offsetHeight) {
        return
      }
      myChart.render()
    }, senior.scrollCfg.interval)
  }
}

const showPage = computed(() => {
  if (view.value.type !== 'table-info') {
    return false
  }
  return state.showPage
})

const handleCurrentChange = pageNum => {
  let extReq = { goPage: pageNum }
  if (chartExtRequest) {
    extReq = { ...extReq, ...chartExtRequest.value }
  }
  const chart = { ...view.value, chartExtRequest: extReq }
  calcData(chart, null, false)
}

const action = param => {
  // 下钻 联动 跳转
  state.pointParam = param
  if (trackMenu.value.length < 2) {
    // 只有一个事件直接调用
    trackClick(trackMenu.value[0])
  } else {
    // 视图关联多个事件
    state.trackBarStyle.left = param.x + 'px'
    state.trackBarStyle.top = param.y + 10 + 'px'
    viewTrack.value.trackButtonClick()
  }
}

const trackClick = trackAction => {
  const param = state.pointParam
  if (!param?.data?.dimensionList) {
    return
  }
  const linkageParam = {
    option: 'linkage',
    name: state.pointParam.data.name,
    viewId: view.value.id,
    dimensionList: state.pointParam.data.dimensionList,
    quotaList: state.pointParam.data.quotaList
  }
  const jumpParam = {
    option: 'jump',
    name: state.pointParam.data.name,
    viewId: view.value.id,
    dimensionList: state.pointParam.data.dimensionList,
    quotaList: state.pointParam.data.quotaList,
    sourceType: state.pointParam.data.sourceType
  }
  switch (trackAction) {
    case 'drill':
      emit('onChartClick', param)
      break
    case 'linkage':
      dvMainStore.addViewTrackFilter(linkageParam)
      break
    case 'jump':
      emit('onJumpClick', jumpParam)
      break
    default:
      break
  }
}

const trackMenu = computed(() => {
  const trackMenuInfo = []
  if (showPosition.value === 'viewDialog') {
    return trackMenuInfo
  }
  let linkageCount = 0
  let jumpCount = 0
  chartData.value?.fields?.forEach(item => {
    const sourceInfo = view.value.id + '#' + item.id
    if (nowPanelTrackInfo.value[sourceInfo]) {
      linkageCount++
    }
    if (nowPanelJumpInfo.value[sourceInfo]) {
      jumpCount++
    }
  })
  jumpCount && view.value?.jumpActive && trackMenuInfo.push('jump')
  linkageCount && view.value?.linkageActive && trackMenuInfo.push('linkage')
  view.value.drillFields.length && trackMenuInfo.push('drill')
  return trackMenuInfo
})

defineExpose({
  calcData,
  renderChart,
  renderChartFromDialog,
  trackMenu
})

let timer
const resize = (width, height) => {
  if (timer) {
    clearTimeout(timer)
  }
  timer = setTimeout(() => {
    myChart?.changeSheetSize(width, height)
    myChart?.render()
    initScroll()
  }, 500)
}
const preSize = [0, 0]
const TOLERANCE = 1
let resizeObserver
onMounted(() => {
  resizeObserver = new ResizeObserver(([entry] = []) => {
    const [size] = entry.borderBoxSize || []
    // 拖动的时候宽高重新计算，误差范围内不重绘，误差先设置为1
    if (!(preSize[0] || preSize[1])) {
      preSize[0] = size.inlineSize
      preSize[1] = size.blockSize
    }
    const widthOffset = Math.abs(size.inlineSize - preSize[0])
    const heightOffset = Math.abs(size.blockSize - preSize[1])
    if (widthOffset < TOLERANCE && heightOffset < TOLERANCE) {
      return
    }
    preSize[0] = size.inlineSize
    preSize[1] = size.blockSize
    resize(size.inlineSize, size.blockSize)
  })

  resizeObserver.observe(document.getElementById(containerId))
})
onBeforeUnmount(() => {
  myChart?.destroy()
  resizeObserver?.disconnect()
  clearInterval(scrollTimer)
})
</script>

<template>
  <div class="canvas-area">
    <view-track-bar
      ref="viewTrack"
      :track-menu="trackMenu"
      class="track-bar"
      :style="state.trackBarStyle"
      @trackClick="trackClick"
    />
    <div v-if="!isError" class="canvas-content">
      <div style="height: 100%" :id="containerId"></div>
    </div>
    <div class="table-page-info" v-if="showPage && !isError">
      <div :style="{ color: pageColor }">共{{ state.pageInfo.total }}条</div>
      <el-pagination
        class="table-page-content"
        layout="prev, pager, next"
        v-model:page-size="state.pageInfo.pageSize"
        v-model:current-page="state.pageInfo.currentPage"
        :style="{ color: pageColor }"
        :pager-count="5"
        :total="state.pageInfo.total"
        @update:current-page="handleCurrentChange"
      />
    </div>
    <chart-error v-if="isError" :err-msg="errMsg" />
  </div>
</template>

<style lang="less" scoped>
.canvas-area {
  z-index: 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
  width: 100%;
  height: 100%;
  .canvas-content {
    flex: 1;
    width: 100%;
    overflow: hidden;
  }
}
.table-page-info {
  margin: 4px;
  height: 20px;
  display: flex;
  width: 100%;
  justify-content: space-between;
  :deep(.table-page-content) {
    button {
      color: inherit;
      background: transparent !important;
    }
    ul li {
      &:not(.is-active) {
        color: inherit;
      }
      background: transparent !important;
    }
  }
}
</style>