<template>
  <div class="traffic-monitor-page">
    <el-card shadow="never" class="control-bar">
      <div class="control-group">
        <el-date-picker
          v-model="filters.timeRange"
          type="datetimerange"
          start-placeholder="开始时间"
          end-placeholder="结束时间"
          :default-time="['00:00:00', '23:59:59']"
          range-separator="至"
          style="width: 320px"
        />
        <el-input v-model="filters.ip" placeholder="过滤 IP 地址" clearable style="width: 160px" />
        <el-input v-model="filters.port" placeholder="端口" clearable style="width: 120px" />
        <el-select v-model="filters.protocol" placeholder="协议" clearable style="width: 140px">
          <el-option v-for="item in protocolOptions" :key="item" :label="item" :value="item" />
        </el-select>
        <el-button type="primary" icon="el-icon-search" @click="applyFilters">应用筛选</el-button>
        <el-button icon="el-icon-refresh" @click="resetFilters">重置</el-button>
      </div>
    </el-card>

    <el-tabs v-model="activeTab" class="monitor-tabs" @tab-click="handleTabChange">
      <el-tab-pane label="流量洞察" name="overview">
        <el-row :gutter="16">
          <el-col :xs="24" :md="16">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>实时总流量波形</span>
                <el-tag type="success" size="mini">刷新间隔 5s</el-tag>
              </div>
              <div ref="overviewChart" class="chart-area" />
            </el-card>
          </el-col>
          <el-col :xs="24" :md="8">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>协议流量占比</span>
                <el-tag type="info" size="mini">当前 5 分钟</el-tag>
              </div>
              <div ref="protocolChart" class="chart-area small" />
            </el-card>
          </el-col>
        </el-row>

        <el-card shadow="hover" class="topn-card">
          <div slot="header" class="card-header">
            <span>TOP N 通信对波形</span>
            <div class="topn-controls">
              <el-switch v-model="autoRefresh" active-text="自动刷新" />
              <el-tag type="info" size="mini">{{ selectedPairLabel }}</el-tag>
            </div>
          </div>
          <div class="topn-layout">
            <el-table
              ref="topnTable"
              :data="displayCommunications"
              border
              height="320"
              highlight-current-row
              class="topn-table"
              @current-change="handlePairSelect"
            >
              <el-table-column label="源地址" min-width="170">
                <template slot-scope="scope">
                  <span>{{ scope.row.source }}</span>
                  <span class="port">:{{ scope.row.sourcePort }}</span>
                </template>
              </el-table-column>
              <el-table-column label="目标地址" min-width="170">
                <template slot-scope="scope">
                  <span>{{ scope.row.target }}</span>
                  <span class="port">:{{ scope.row.targetPort }}</span>
                </template>
              </el-table-column>
              <el-table-column prop="protocol" label="协议" width="100">
                <template slot-scope="scope">
                  <el-tag :type="protocolTag(scope.row.protocol)" size="mini">{{ scope.row.protocol }}</el-tag>
                </template>
              </el-table-column>
              <el-table-column prop="pps" label="包率 (pps)" width="140" />
              <el-table-column prop="bandwidth" label="带宽 (Mbps)" width="160" />
              <el-table-column prop="deviation" label="偏离基线" width="120">
                <template slot-scope="scope">
                  <span :class="['trend', scope.row.deviation >= 0 ? 'up' : 'down']">
                    {{ scope.row.deviation >= 0 ? '+' : '' }}{{ scope.row.deviation }}%
                  </span>
                </template>
              </el-table-column>
              <el-table-column prop="lastSeen" label="最近观测" min-width="180">
                <template slot-scope="scope">
                  <span class="last-seen">{{ scope.row.lastSeen }}</span>
                </template>
              </el-table-column>
            </el-table>
            <div class="pair-chart">
              <div class="pair-chart-title">
                <span>通信对流量波形</span>
                <el-tag v-if="selectedCommunication" type="success" size="mini">{{ pairChartSubtitle }}</el-tag>
                <el-tag v-else type="info" size="mini">选择通信对以查看</el-tag>
              </div>
              <div ref="pairTrendChart" class="chart-area pair" />
            </div>
          </div>
        </el-card>

        <el-row :gutter="16" class="analysis-row">
          <el-col :xs="24" :md="12">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>时间序列分析</span>
                <el-select v-model="analysisGranularity" size="mini" style="width: 120px">
                  <el-option label="分钟" value="minute" />
                  <el-option label="小时" value="hour" />
                  <el-option label="天" value="day" />
                </el-select>
              </div>
              <div ref="timeSeriesChart" class="chart-area" />
            </el-card>
          </el-col>
          <el-col :xs="24" :md="12">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>超常规调用分析</span>
                <el-tag type="danger" size="mini">偏离告警</el-tag>
              </div>
              <div ref="deviationChart" class="chart-area" />
            </el-card>
          </el-col>
        </el-row>
      </el-tab-pane>

      <el-tab-pane label="异常检测" name="anomaly">
        <el-row :gutter="16">
          <el-col :xs="24" :md="14">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>异常流量模式</span>
                <el-tag type="danger" size="mini">实时告警</el-tag>
              </div>
              <div ref="anomalyChart" class="chart-area" />
            </el-card>
          </el-col>
          <el-col :xs="24" :md="10">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>策略响应</span>
                <el-button type="text" size="mini">调整策略</el-button>
              </div>
              <el-timeline>
                <el-timeline-item
                  v-for="action in mitigationActions"
                  :key="action.id"
                  :timestamp="action.time"
                  :type="action.type"
                >
                  <div class="mitigation-item">
                    <div class="mitigation-title">{{ action.title }}</div>
                    <p class="mitigation-desc">{{ action.description }}</p>
                  </div>
                </el-timeline-item>
              </el-timeline>
            </el-card>
          </el-col>
        </el-row>
        <el-card shadow="hover" class="table-card">
          <div slot="header" class="card-header">
            <span>异常事件清单</span>
            <el-tag type="info" size="mini">近 24 小时</el-tag>
          </div>
          <el-table :data="anomalyEvents" border height="260">
            <el-table-column prop="time" label="时间" width="160" />
            <el-table-column prop="type" label="类型" width="150">
              <template slot-scope="scope">
                <el-tag :type="scope.row.tag" size="mini">{{ scope.row.type }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="source" label="源" min-width="160" />
            <el-table-column prop="description" label="描述" min-width="240" />
            <el-table-column prop="status" label="状态" width="120">
              <template slot-scope="scope">
                <el-tag :type="scope.row.statusTag" size="mini">{{ scope.row.status }}</el-tag>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-tab-pane>
    </el-tabs>
  </div>
</template>

<script>
import * as echarts from 'echarts'

export default {
  name: 'TrafficMonitor',
  data() {
    const toTimestamp = timeString => new Date(timeString).getTime()
    const pad = value => (value < 10 ? `0${value}` : `${value}`)
    const formatDateTime = timestamp => {
      const date = new Date(timestamp)
      return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}`
    }
    const computeBaseline = (bandwidth, deviation) =>
      Number((bandwidth / (1 + deviation / 100)).toFixed(1))

    return {
      filters: {
        timeRange: [],
        ip: '',
        port: '',
        protocol: ''
      },
      activeTab: 'overview',
      autoRefresh: true,
      analysisGranularity: 'hour',
      rawCommunications: [
        {
          id: 'pair-1',
          source: '10.1.2.11',
          sourcePort: 8443,
          target: '172.16.0.21',
          targetPort: 443,
          protocol: 'HTTPS',
          pps: 4210,
          bandwidth: 86,
          baseline: computeBaseline(86, 18),
          ppsRatio: 4210 / 86,
          deviation: 18,
          window: [toTimestamp('2024-10-19T14:05:00'), toTimestamp('2024-10-19T14:50:00')],
          lastSeen: formatDateTime(toTimestamp('2024-10-19T14:50:00')),
          timeline: [
            { time: '14:05', timestamp: toTimestamp('2024-10-19T14:05:00'), value: 62 },
            { time: '14:10', timestamp: toTimestamp('2024-10-19T14:10:00'), value: 65 },
            { time: '14:15', timestamp: toTimestamp('2024-10-19T14:15:00'), value: 69 },
            { time: '14:20', timestamp: toTimestamp('2024-10-19T14:20:00'), value: 73 },
            { time: '14:25', timestamp: toTimestamp('2024-10-19T14:25:00'), value: 80 },
            { time: '14:30', timestamp: toTimestamp('2024-10-19T14:30:00'), value: 88 },
            { time: '14:35', timestamp: toTimestamp('2024-10-19T14:35:00'), value: 92 },
            { time: '14:40', timestamp: toTimestamp('2024-10-19T14:40:00'), value: 95 },
            { time: '14:45', timestamp: toTimestamp('2024-10-19T14:45:00'), value: 91 },
            { time: '14:50', timestamp: toTimestamp('2024-10-19T14:50:00'), value: 86 }
          ]
        },
        {
          id: 'pair-2',
          source: '10.1.2.14',
          sourcePort: 8080,
          target: '172.16.0.34',
          targetPort: 80,
          protocol: 'HTTP',
          pps: 3560,
          bandwidth: 42,
          baseline: computeBaseline(42, -6),
          ppsRatio: 3560 / 42,
          deviation: -6,
          window: [toTimestamp('2024-10-19T14:05:00'), toTimestamp('2024-10-19T14:50:00')],
          lastSeen: formatDateTime(toTimestamp('2024-10-19T14:50:00')),
          timeline: [
            { time: '14:05', timestamp: toTimestamp('2024-10-19T14:05:00'), value: 44 },
            { time: '14:10', timestamp: toTimestamp('2024-10-19T14:10:00'), value: 45 },
            { time: '14:15', timestamp: toTimestamp('2024-10-19T14:15:00'), value: 43 },
            { time: '14:20', timestamp: toTimestamp('2024-10-19T14:20:00'), value: 40 },
            { time: '14:25', timestamp: toTimestamp('2024-10-19T14:25:00'), value: 39 },
            { time: '14:30', timestamp: toTimestamp('2024-10-19T14:30:00'), value: 37 },
            { time: '14:35', timestamp: toTimestamp('2024-10-19T14:35:00'), value: 36 },
            { time: '14:40', timestamp: toTimestamp('2024-10-19T14:40:00'), value: 34 },
            { time: '14:45', timestamp: toTimestamp('2024-10-19T14:45:00'), value: 33 },
            { time: '14:50', timestamp: toTimestamp('2024-10-19T14:50:00'), value: 32 }
          ]
        },
        {
          id: 'pair-3',
          source: '10.1.3.20',
          sourcePort: 1883,
          target: '172.16.0.82',
          targetPort: 3883,
          protocol: 'MQTT',
          pps: 2112,
          bandwidth: 33,
          baseline: computeBaseline(33, 12),
          ppsRatio: 2112 / 33,
          deviation: 12,
          window: [toTimestamp('2024-10-19T14:05:00'), toTimestamp('2024-10-19T14:50:00')],
          lastSeen: formatDateTime(toTimestamp('2024-10-19T14:50:00')),
          timeline: [
            { time: '14:05', timestamp: toTimestamp('2024-10-19T14:05:00'), value: 22 },
            { time: '14:10', timestamp: toTimestamp('2024-10-19T14:10:00'), value: 25 },
            { time: '14:15', timestamp: toTimestamp('2024-10-19T14:15:00'), value: 26 },
            { time: '14:20', timestamp: toTimestamp('2024-10-19T14:20:00'), value: 28 },
            { time: '14:25', timestamp: toTimestamp('2024-10-19T14:25:00'), value: 29 },
            { time: '14:30', timestamp: toTimestamp('2024-10-19T14:30:00'), value: 31 },
            { time: '14:35', timestamp: toTimestamp('2024-10-19T14:35:00'), value: 33 },
            { time: '14:40', timestamp: toTimestamp('2024-10-19T14:40:00'), value: 36 },
            { time: '14:45', timestamp: toTimestamp('2024-10-19T14:45:00'), value: 35 },
            { time: '14:50', timestamp: toTimestamp('2024-10-19T14:50:00'), value: 34 }
          ]
        },
        {
          id: 'pair-4',
          source: '10.2.1.51',
          sourcePort: 2404,
          target: '172.16.0.99',
          targetPort: 2404,
          protocol: 'IEC104',
          pps: 1920,
          bandwidth: 28,
          baseline: computeBaseline(28, 5),
          ppsRatio: 1920 / 28,
          deviation: 5,
          window: [toTimestamp('2024-10-19T14:05:00'), toTimestamp('2024-10-19T14:50:00')],
          lastSeen: formatDateTime(toTimestamp('2024-10-19T14:50:00')),
          timeline: [
            { time: '14:05', timestamp: toTimestamp('2024-10-19T14:05:00'), value: 21 },
            { time: '14:10', timestamp: toTimestamp('2024-10-19T14:10:00'), value: 22 },
            { time: '14:15', timestamp: toTimestamp('2024-10-19T14:15:00'), value: 24 },
            { time: '14:20', timestamp: toTimestamp('2024-10-19T14:20:00'), value: 26 },
            { time: '14:25', timestamp: toTimestamp('2024-10-19T14:25:00'), value: 27 },
            { time: '14:30', timestamp: toTimestamp('2024-10-19T14:30:00'), value: 28 },
            { time: '14:35', timestamp: toTimestamp('2024-10-19T14:35:00'), value: 29 },
            { time: '14:40', timestamp: toTimestamp('2024-10-19T14:40:00'), value: 28 },
            { time: '14:45', timestamp: toTimestamp('2024-10-19T14:45:00'), value: 27 },
            { time: '14:50', timestamp: toTimestamp('2024-10-19T14:50:00'), value: 26 }
          ]
        },
        {
          id: 'pair-5',
          source: '10.2.5.18',
          sourcePort: 502,
          target: '172.16.0.45',
          targetPort: 1502,
          protocol: 'Modbus',
          pps: 1422,
          bandwidth: 21,
          baseline: computeBaseline(21, -3),
          ppsRatio: 1422 / 21,
          deviation: -3,
          window: [toTimestamp('2024-10-19T14:05:00'), toTimestamp('2024-10-19T14:50:00')],
          lastSeen: formatDateTime(toTimestamp('2024-10-19T14:50:00')),
          timeline: [
            { time: '14:05', timestamp: toTimestamp('2024-10-19T14:05:00'), value: 23 },
            { time: '14:10', timestamp: toTimestamp('2024-10-19T14:10:00'), value: 23 },
            { time: '14:15', timestamp: toTimestamp('2024-10-19T14:15:00'), value: 22 },
            { time: '14:20', timestamp: toTimestamp('2024-10-19T14:20:00'), value: 21 },
            { time: '14:25', timestamp: toTimestamp('2024-10-19T14:25:00'), value: 20 },
            { time: '14:30', timestamp: toTimestamp('2024-10-19T14:30:00'), value: 20 },
            { time: '14:35', timestamp: toTimestamp('2024-10-19T14:35:00'), value: 19 },
            { time: '14:40', timestamp: toTimestamp('2024-10-19T14:40:00'), value: 18 },
            { time: '14:45', timestamp: toTimestamp('2024-10-19T14:45:00'), value: 18 },
            { time: '14:50', timestamp: toTimestamp('2024-10-19T14:50:00'), value: 17 }
          ]
        }
      ],
      displayCommunications: [],
      selectedCommunication: null,
      autoRefreshTimer: null,
      callDeviation: [
        { name: '负荷预测 → 调度主站', baseline: 120, actual: 168 },
        { name: '场站 AGC → AGC 集中器', baseline: 96, actual: 142 },
        { name: '交易撮合 → 结算中台', baseline: 88, actual: 103 },
        { name: '边缘监测 → 实时数据湖', baseline: 64, actual: 88 },
        { name: '运维工单 → 安全域代理', baseline: 52, actual: 49 }
      ],
      anomalyEvents: [
        { time: '2024-10-19 14:44', type: 'DDoS 攻击', tag: 'danger', source: '北部探针 #3', description: '检测到大量 SYN 洪水攻击，已启用丢弃策略', status: '已阻断', statusTag: 'success' },
        { time: '2024-10-19 14:18', type: '端口扫描', tag: 'warning', source: '南部探针 #1', description: '内外网边界发现高频端口探测行为', status: '观察中', statusTag: 'warning' },
        { time: '2024-10-19 13:57', type: '异常大流量', tag: 'info', source: '东部探针 #2', description: '归档服务上传数据超出阈值，已通知负责人', status: '待确认', statusTag: 'info' }
      ],
      mitigationActions: [
        { id: 1, title: '触发自动限速策略', description: '对攻击源 10.1.2.254 启动 5 分钟带宽限制', time: '13:42', type: 'danger' },
        { id: 2, title: '联动防火墙封禁', description: '推送 6 条封禁规则至边界防火墙', time: '13:44', type: 'warning' },
        { id: 3, title: '通知服务负责人', description: '消息队列服务负责人已确认异常波动', time: '12:02', type: 'success' }
      ],
      charts: {
        overview: null,
        protocol: null,
        pairTrend: null,
        timeSeries: null,
        deviation: null,
        anomaly: null
      }
    }
  },
  computed: {
    protocolOptions: function() {
      const options = Array.from(new Set(this.rawCommunications.map(item => item.protocol)))
      return options.sort()
    },
    selectedPairLabel: function() {
      if (!this.selectedCommunication) {
        return '未选择通信对'
      }
      const pair = this.selectedCommunication
      return `当前：${pair.source}:${pair.sourcePort} -> ${pair.target}:${pair.targetPort}`
    },
    pairChartSubtitle: function() {
      if (!this.selectedCommunication) {
        return ''
      }
      const pair = this.selectedCommunication
      return `${pair.protocol} | ${pair.source}:${pair.sourcePort} -> ${pair.target}:${pair.targetPort}`
    }
  },
  watch: {
    analysisGranularity() {
      this.updateTimeSeriesChart()
    },
    autoRefresh(value) {
      if (value) {
        this.startAutoRefresh()
      } else {
        this.stopAutoRefresh()
      }
    },
    displayCommunications: {
      handler() {
        this.$nextTick(() => {
          this.ensureSelection()
          this.refreshCharts()
        })
      },
      deep: true
    }
  },
  mounted() {
    this.filterCommunications()
    this.selectedCommunication = this.displayCommunications[0] || null
    this.$nextTick(() => {
      this.initCharts()
      window.addEventListener('resize', this.handleResize)
      this.ensureSelection()
      if (this.autoRefresh) {
        this.startAutoRefresh()
      }
    })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
    this.stopAutoRefresh()
    this.disposeCharts()
  },
  methods: {
    initCharts() {
      this.initOverviewChart()
      this.initProtocolChart()
      this.initPairTrendChart()
      this.initTimeSeriesChart()
      this.initDeviationChart()
      this.initAnomalyChart()
    },
    initOverviewChart() {
      if (!this.$refs.overviewChart) return
      this.charts.overview = echarts.init(this.$refs.overviewChart)
      this.updateOverviewChart()
    },
    updateOverviewChart() {
      if (!this.charts.overview) return
      const aggregated = this.getAggregatedTimeline()
      if (!aggregated.length) {
        this.setChartEmptyState(this.charts.overview, '暂无流量数据')
        return
      }
      const categories = aggregated.map(item => item.label)
      const totals = aggregated.map(item => Number(item.value.toFixed(2)))
      this.charts.overview.clear()
      this.charts.overview.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 32, left: 32, right: 16, bottom: 32 },
        xAxis: {
          type: 'category',
          data: categories,
          boundaryGap: false,
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          }
        },
        yAxis: {
          type: 'value',
          name: 'Mbps',
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          },
          splitLine: {
            lineStyle: {
              type: 'dashed',
              color: '#dcdfe6'
            }
          }
        },
        series: [
          {
            name: '聚合流量',
            type: 'line',
            smooth: true,
            symbol: 'circle',
            symbolSize: 6,
            areaStyle: {
              color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                { offset: 0, color: 'rgba(64, 158, 255, 0.4)' },
                { offset: 1, color: 'rgba(64, 158, 255, 0.05)' }
              ])
            },
            lineStyle: { width: 2, color: '#409EFF' },
            itemStyle: { color: '#409EFF' },
            data: totals
          }
        ]
      })
    },
    initProtocolChart() {
      if (!this.$refs.protocolChart) return
      this.charts.protocol = echarts.init(this.$refs.protocolChart)
      this.updateProtocolChart()
    },
    updateProtocolChart() {
      if (!this.charts.protocol) return
      const totals = this.displayCommunications.reduce((map, item) => {
        const current = map.get(item.protocol) || 0
        map.set(item.protocol, current + item.bandwidth)
        return map
      }, new Map())
      if (!totals.size) {
        this.setChartEmptyState(this.charts.protocol, '暂无协议分布')
        return
      }
      const data = Array.from(totals.entries()).map(([name, value]) => ({
        name,
        value: Number(value.toFixed(1))
      }))
      this.charts.protocol.clear()
      this.charts.protocol.setOption({
        tooltip: { trigger: 'item' },
        legend: {
          bottom: 0,
          textStyle: {
            color: '#606266'
          }
        },
        series: [
          {
            type: 'pie',
            radius: ['35%', '65%'],
            roseType: 'radius',
            itemStyle: {
              borderRadius: 6,
              borderColor: '#fff',
              borderWidth: 2
            },
            label: { formatter: '{b}: {d}%' },
            data
          }
        ]
      })
    },
    initPairTrendChart() {
      if (!this.$refs.pairTrendChart) return
      this.charts.pairTrend = echarts.init(this.$refs.pairTrendChart)
      this.updatePairTrendChart()
    },
    updatePairTrendChart() {
      if (!this.charts.pairTrend) return
      if (!this.selectedCommunication || !this.selectedCommunication.timeline.length) {
        this.setChartEmptyState(this.charts.pairTrend, '选择通信对以查看波形')
        return
      }
      const categories = this.selectedCommunication.timeline.map(item => item.time)
      const values = this.selectedCommunication.timeline.map(item => Number(item.value.toFixed(2)))
      this.charts.pairTrend.clear()
      this.charts.pairTrend.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 24, left: 32, right: 16, bottom: 32 },
        xAxis: {
          type: 'category',
          data: categories,
          boundaryGap: false,
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          }
        },
        yAxis: {
          type: 'value',
          name: 'Mbps',
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          },
          splitLine: {
            lineStyle: {
              type: 'dashed',
              color: '#ebeef5'
            }
          }
        },
        series: [
          {
            name: '通信对流量',
            type: 'line',
            smooth: true,
            symbol: 'circle',
            symbolSize: 6,
            lineStyle: { width: 2, color: '#67C23A' },
            itemStyle: { color: '#67C23A' },
            areaStyle: {
              color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                { offset: 0, color: 'rgba(103, 194, 58, 0.35)' },
                { offset: 1, color: 'rgba(103, 194, 58, 0.05)' }
              ])
            },
            data: values
          }
        ]
      })
    },
    initTimeSeriesChart() {
      if (!this.$refs.timeSeriesChart) return
      this.charts.timeSeries = echarts.init(this.$refs.timeSeriesChart)
      this.updateTimeSeriesChart()
    },
    updateTimeSeriesChart() {
      if (!this.charts.timeSeries) return
      const preset = this.computeTimeSeries(this.analysisGranularity)
      if (!preset.categories.length) {
        this.setChartEmptyState(this.charts.timeSeries, '暂无时间序列数据')
        return
      }
      this.charts.timeSeries.clear()
      this.charts.timeSeries.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 32, left: 36, right: 16, bottom: 32 },
        xAxis: {
          type: 'category',
          data: preset.categories,
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          }
        },
        yAxis: {
          type: 'value',
          name: 'Mbps',
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          },
          splitLine: {
            lineStyle: {
              type: 'dashed',
              color: '#ebeef5'
            }
          }
        },
        series: [
          {
            name: '区间流量',
            type: 'line',
            smooth: true,
            symbol: 'circle',
            itemStyle: { color: '#67C23A' },
            lineStyle: { color: '#67C23A', width: 3 },
            data: preset.values
          }
        ]
      })
    },
    initDeviationChart() {
      if (!this.$refs.deviationChart) return
      this.charts.deviation = echarts.init(this.$refs.deviationChart)
      const categories = this.callDeviation.map(item => item.name)
      const baseline = this.callDeviation.map(item => item.baseline)
      const actual = this.callDeviation.map(item => item.actual)
      this.charts.deviation.setOption({
        tooltip: {
          trigger: 'axis',
          axisPointer: {
            type: 'shadow'
          }
        },
        legend: { top: 0, data: ['基线', '实际'] },
        grid: { top: 52, left: 160, right: 16, bottom: 24 },
        xAxis: {
          type: 'value',
          name: 'Mbps',
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          }
        },
        yAxis: {
          type: 'category',
          data: categories,
          inverse: true,
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          }
        },
        series: [
          {
            name: '基线',
            type: 'bar',
            barWidth: 14,
            itemStyle: { color: '#B3C0D1' },
            data: baseline
          },
          {
            name: '实际',
            type: 'bar',
            barWidth: 14,
            itemStyle: { color: '#F56C6C' },
            data: actual,
            markPoint: {
              symbol: 'pin',
              symbolSize: 38,
              label: {
                color: '#fff',
                formatter: ({ value }) => `${value}`
              },
              data: this.callDeviation.reduce((points, item, index) => {
                if (item.actual > item.baseline) {
                  points.push({
                    name: '偏离',
                    value: `+${item.actual - item.baseline}`,
                    yAxis: categories[index],
                    xAxis: item.actual
                  })
                }
                return points
              }, [])
            }
          }
        ]
      })
    },
    initAnomalyChart() {
      if (!this.$refs.anomalyChart) return
      this.charts.anomaly = echarts.init(this.$refs.anomalyChart)
      const categories = ['10:00', '11:00', '12:00', '13:00', '14:00']
      this.charts.anomaly.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 32, left: 36, right: 16, bottom: 32 },
        legend: { top: 0 },
        xAxis: {
          type: 'category',
          data: categories,
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          }
        },
        yAxis: {
          type: 'value',
          name: '事件数',
          axisLine: {
            lineStyle: {
              color: '#909399'
            }
          },
          splitLine: {
            lineStyle: {
              type: 'dashed',
              color: '#ebeef5'
            }
          }
        },
        series: [
          {
            name: 'DDoS',
            type: 'line',
            smooth: true,
            data: [3, 5, 4, 6, 5],
            itemStyle: { color: '#F56C6C' }
          },
          {
            name: '端口扫描',
            type: 'line',
            smooth: true,
            data: [2, 2, 3, 4, 3],
            itemStyle: { color: '#E6A23C' }
          },
          {
            name: '异常大流量',
            type: 'line',
            smooth: true,
            data: [1, 2, 1, 3, 2],
            itemStyle: { color: '#409EFF' }
          }
        ]
      })
    },
    applyFilters() {
      this.filterCommunications()
      this.$message.success('筛选条件已应用')
    },
    resetFilters() {
      this.filters = {
        timeRange: [],
        ip: '',
        port: '',
        protocol: ''
      }
      this.filterCommunications()
      this.$message.info('筛选条件已重置')
    },
    protocolTag(protocol) {
      switch (protocol) {
        case 'HTTPS':
          return 'success'
        case 'HTTP':
          return 'info'
        case 'MQTT':
          return 'warning'
        case 'IEC104':
          return 'danger'
        default:
          return 'default'
      }
    },
    handlePairSelect(row) {
      this.selectedCommunication = row || null
      this.updatePairTrendChart()
    },
    handleResize() {
      Object.keys(this.charts).forEach(key => {
        if (this.charts[key]) {
          this.charts[key].resize()
        }
      })
    },
    disposeCharts() {
      Object.keys(this.charts).forEach(key => {
        if (this.charts[key]) {
          this.charts[key].dispose()
          this.charts[key] = null
        }
      })
    },
    handleTabChange() {
      this.$nextTick(() => {
        this.handleResize()
      })
    },
    filterCommunications() {
      const { ip, port, protocol, timeRange } = this.filters
      const [start, end] =
        timeRange && timeRange.length === 2 ? timeRange.map(item => item.getTime()) : [null, null]
      const ipKeyword = ip.trim()
      const portKeyword = port.trim()
      const filtered = this.rawCommunications.filter(item => {
        const ipMatch =
          !ipKeyword || item.source.includes(ipKeyword) || item.target.includes(ipKeyword)
        const portMatch =
          !portKeyword ||
          item.sourcePort.toString() === portKeyword ||
          item.targetPort.toString() === portKeyword
        const protocolMatch = !protocol || item.protocol === protocol
        const timeMatch =
          !start || !end
            ? true
            : item.timeline.some(point => point.timestamp >= start && point.timestamp <= end)
        return ipMatch && portMatch && protocolMatch && timeMatch
      })
      const sorted = filtered.slice().sort((a, b) => b.bandwidth - a.bandwidth)
      this.displayCommunications = sorted.slice(0, 10)
    },
    ensureSelection() {
      if (!this.$refs.topnTable) {
        return
      }
      if (!this.displayCommunications.length) {
        this.$refs.topnTable.setCurrentRow()
        this.selectedCommunication = null
        this.updatePairTrendChart()
        return
      }
      const currentId = this.selectedCommunication ? this.selectedCommunication.id : null
      const candidate = currentId
        ? this.displayCommunications.find(item => item.id === currentId)
        : null
      this.selectedCommunication = candidate || this.displayCommunications[0]
      this.$nextTick(() => {
        if (this.$refs.topnTable) {
          this.$refs.topnTable.setCurrentRow(this.selectedCommunication)
        }
        this.updatePairTrendChart()
      })
    },
    refreshCharts() {
      this.updateOverviewChart()
      this.updateProtocolChart()
      this.updatePairTrendChart()
      this.updateTimeSeriesChart()
    },
    startAutoRefresh() {
      this.stopAutoRefresh()
      this.autoRefreshTimer = setInterval(() => {
        this.simulateRealtimeUpdate()
      }, 5000)
    },
    stopAutoRefresh() {
      if (this.autoRefreshTimer) {
        clearInterval(this.autoRefreshTimer)
        this.autoRefreshTimer = null
      }
    },
    simulateRealtimeUpdate() {
      const step = 5 * 60 * 1000
      this.rawCommunications.forEach(item => {
        const lastPoint = item.timeline[item.timeline.length - 1]
        const nextTimestamp = (lastPoint ? lastPoint.timestamp : Date.now()) + step
        const drift = 0.92 + Math.random() * 0.18
        const baseValue = lastPoint ? lastPoint.value : item.bandwidth
        const nextValue = Math.max(5, Number((baseValue * drift).toFixed(1)))
        item.timeline.push({
          time: this.formatTime(nextTimestamp),
          timestamp: nextTimestamp,
          value: nextValue
        })
        if (item.timeline.length > 12) {
          item.timeline.shift()
        }
        item.window = [item.timeline[0].timestamp, item.timeline[item.timeline.length - 1].timestamp]
        item.bandwidth = nextValue
        item.pps = Math.round(item.bandwidth * item.ppsRatio)
        item.deviation = Number((((item.bandwidth - item.baseline) / item.baseline) * 100).toFixed(1))
        item.lastSeen = this.formatDateTime(item.window[1])
      })
      this.filterCommunications()
    },
    getAggregatedTimeline() {
      const buckets = new Map()
      this.displayCommunications.forEach(item => {
        item.timeline.forEach(point => {
          const current = buckets.get(point.timestamp) || { timestamp: point.timestamp, label: point.time, value: 0 }
          current.value += point.value
          current.label = point.time
          buckets.set(point.timestamp, current)
        })
      })
      return Array.from(buckets.values()).sort((a, b) => a.timestamp - b.timestamp)
    },
    computeTimeSeries(granularity) {
      const aggregated = this.getAggregatedTimeline()
      if (!aggregated.length) {
        return { categories: [], values: [] }
      }
      const buckets = new Map()
      aggregated.forEach(item => {
        const { key, label } = this.getTimeBucketKey(item.timestamp, granularity)
        if (!buckets.has(key)) {
          buckets.set(key, { label, value: 0 })
        }
        const record = buckets.get(key)
        record.value += item.value
      })
      const sorted = Array.from(buckets.entries()).sort((a, b) => a[0] - b[0])
      return {
        categories: sorted.map(([, entry]) => entry.label),
        values: sorted.map(([, entry]) => Number(entry.value.toFixed(2)))
      }
    },
    getTimeBucketKey(timestamp, granularity) {
      const date = new Date(timestamp)
      if (granularity === 'minute') {
        const key = new Date(
          date.getFullYear(),
          date.getMonth(),
          date.getDate(),
          date.getHours(),
          date.getMinutes()
        ).getTime()
        return { key, label: this.formatTime(key) }
      }
      if (granularity === 'hour') {
        const key = new Date(date.getFullYear(), date.getMonth(), date.getDate(), date.getHours()).getTime()
        return { key, label: `${this.pad(date.getHours())}:00` }
      }
      const dayLabels = ['周日', '周一', '周二', '周三', '周四', '周五', '周六']
      const day = date.getDay()
      return { key: day, label: dayLabels[day] }
    },
    formatTime(timestamp) {
      const date = new Date(timestamp)
      return `${this.pad(date.getHours())}:${this.pad(date.getMinutes())}`
    },
    formatDateTime(timestamp) {
      const date = new Date(timestamp)
      return `${date.getFullYear()}-${this.pad(date.getMonth() + 1)}-${this.pad(date.getDate())} ${this.pad(date.getHours())}:${this.pad(date.getMinutes())}`
    },
    pad(value) {
      return value < 10 ? `0${value}` : `${value}`
    },
    setChartEmptyState(chart, message) {
      if (!chart) {
        return
      }
      chart.clear()
      chart.setOption({
        title: {
          text: message,
          left: 'center',
          top: 'middle',
          textStyle: {
            color: '#909399',
            fontSize: 14,
            fontWeight: 500
          }
        }
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.traffic-monitor-page {
  padding: 16px;
  background: #f5f7fa;
  min-height: 100%;
}

.control-bar {
  margin-bottom: 16px;
  border-radius: 12px;
  .control-group {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
  }
}

.monitor-tabs ::v-deep .el-tabs__header {
  margin-bottom: 16px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 600;
}

.chart-area {
  width: 100%;
  height: 280px;
  &.small {
    height: 260px;
  }
  &.pair {
    height: 320px;
  }
}

.table-card {
  margin-top: 16px;
}

.topn-card {
  margin-top: 16px;
  border-radius: 12px;
}

.topn-controls {
  display: flex;
  align-items: center;
  gap: 8px;
}

.topn-layout {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.topn-table {
  flex: 1 1 100%;
}

.pair-chart {
  flex: 1 1 100%;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.pair-chart-title {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 600;
}

.analysis-row {
  margin-top: 16px;
}

.trend {
  font-weight: 600;
  &.up {
    color: #f56c6c;
  }
  &.down {
    color: #67c23a;
  }
}

.port {
  margin-left: 4px;
  color: #409eff;
  font-weight: 600;
}

.last-seen {
  font-size: 13px;
  color: #606266;
}

.mitigation-item {
  .mitigation-title {
    font-weight: 600;
    margin-bottom: 4px;
  }
  .mitigation-desc {
    font-size: 13px;
    color: #606266;
    margin: 0;
  }
}

@media (max-width: 1024px) {
  .chart-area {
    height: 240px;
  }
  .chart-area.pair {
    height: 260px;
  }
}

@media (min-width: 1025px) {
  .topn-layout {
    flex-direction: row;
  }
  .topn-table {
    flex: 1 1 54%;
  }
  .pair-chart {
    flex: 1 1 46%;
  }
}
</style>
