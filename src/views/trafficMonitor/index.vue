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
          <el-option v-for="item in protocols" :key="item" :label="item" :value="item" />
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
              :data="topCommunications"
              border
              height="320"
              highlight-current-row
              class="topn-table"
              @current-change="handlePairSelect"
            >
              <el-table-column prop="source" label="源地址" min-width="150" />
              <el-table-column prop="target" label="目标地址" min-width="150" />
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
    return {
      filters: {
        timeRange: [],
        ip: '',
        port: '',
        protocol: ''
      },
      protocols: ['HTTP', 'HTTPS', 'MQTT', 'IEC104', 'Modbus'],
      activeTab: 'overview',
      autoRefresh: true,
      analysisGranularity: 'hour',
      topCommunications: [
        {
          source: '10.1.2.11',
          target: '172.16.0.21',
          protocol: 'HTTPS',
          pps: 4210,
          bandwidth: 86,
          deviation: 18,
          timeline: [
            { time: '09:05', value: 62 },
            { time: '09:10', value: 65 },
            { time: '09:15', value: 69 },
            { time: '09:20', value: 73 },
            { time: '09:25', value: 80 },
            { time: '09:30', value: 88 },
            { time: '09:35', value: 92 },
            { time: '09:40', value: 95 },
            { time: '09:45', value: 91 },
            { time: '09:50', value: 86 }
          ]
        },
        {
          source: '10.1.2.14',
          target: '172.16.0.34',
          protocol: 'HTTP',
          pps: 3560,
          bandwidth: 42,
          deviation: -6,
          timeline: [
            { time: '09:05', value: 44 },
            { time: '09:10', value: 45 },
            { time: '09:15', value: 43 },
            { time: '09:20', value: 40 },
            { time: '09:25', value: 39 },
            { time: '09:30', value: 37 },
            { time: '09:35', value: 36 },
            { time: '09:40', value: 34 },
            { time: '09:45', value: 33 },
            { time: '09:50', value: 32 }
          ]
        },
        {
          source: '10.1.3.20',
          target: '172.16.0.82',
          protocol: 'MQTT',
          pps: 2112,
          bandwidth: 33,
          deviation: 12,
          timeline: [
            { time: '09:05', value: 22 },
            { time: '09:10', value: 25 },
            { time: '09:15', value: 26 },
            { time: '09:20', value: 28 },
            { time: '09:25', value: 29 },
            { time: '09:30', value: 31 },
            { time: '09:35', value: 33 },
            { time: '09:40', value: 36 },
            { time: '09:45', value: 35 },
            { time: '09:50', value: 34 }
          ]
        },
        {
          source: '10.2.1.51',
          target: '172.16.0.99',
          protocol: 'IEC104',
          pps: 1920,
          bandwidth: 28,
          deviation: 5,
          timeline: [
            { time: '09:05', value: 21 },
            { time: '09:10', value: 22 },
            { time: '09:15', value: 24 },
            { time: '09:20', value: 26 },
            { time: '09:25', value: 27 },
            { time: '09:30', value: 28 },
            { time: '09:35', value: 29 },
            { time: '09:40', value: 28 },
            { time: '09:45', value: 27 },
            { time: '09:50', value: 26 }
          ]
        },
        {
          source: '10.2.5.18',
          target: '172.16.0.45',
          protocol: 'Modbus',
          pps: 1422,
          bandwidth: 21,
          deviation: -3,
          timeline: [
            { time: '09:05', value: 23 },
            { time: '09:10', value: 23 },
            { time: '09:15', value: 22 },
            { time: '09:20', value: 21 },
            { time: '09:25', value: 20 },
            { time: '09:30', value: 20 },
            { time: '09:35', value: 19 },
            { time: '09:40', value: 18 },
            { time: '09:45', value: 18 },
            { time: '09:50', value: 17 }
          ]
        }
      ],
      selectedCommunication: null,
      timeSeriesPresets: {
        minute: {
          categories: ['09:41', '09:42', '09:43', '09:44', '09:45', '09:46', '09:47', '09:48', '09:49', '09:50'],
          values: [82, 84, 87, 86, 88, 92, 95, 97, 93, 90]
        },
        hour: {
          categories: ['01:00', '03:00', '05:00', '07:00', '09:00', '11:00', '13:00'],
          values: [160, 180, 210, 245, 268, 256, 242]
        },
        day: {
          categories: ['周一', '周二', '周三', '周四', '周五', '周六', '周日'],
          values: [1320, 1460, 1580, 1660, 1730, 1420, 1180]
        }
      },
      callDeviation: [
        { name: '负荷预测 → 调度主站', baseline: 120, actual: 168 },
        { name: '场站 AGC → AGC 集中器', baseline: 96, actual: 142 },
        { name: '交易撮合 → 结算中台', baseline: 88, actual: 103 },
        { name: '边缘监测 → 实时数据湖', baseline: 64, actual: 88 },
        { name: '运维工单 → 安全域代理', baseline: 52, actual: 49 }
      ],
      anomalyEvents: [
        { time: '2024-08-12 09:44', type: 'DDoS 攻击', tag: 'danger', source: '北部探针 #3', description: '检测到大量 SYN 洪水攻击，已启用丢弃策略', status: '已阻断', statusTag: 'success' },
        { time: '2024-08-12 09:12', type: '端口扫描', tag: 'warning', source: '南部探针 #1', description: '内外网边界发现高频端口探测行为', status: '观察中', statusTag: 'warning' },
        { time: '2024-08-12 08:55', type: '异常大流量', tag: 'info', source: '东部探针 #2', description: '归档服务上传数据超出阈值，已通知负责人', status: '待确认', statusTag: 'info' }
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
    selectedPairLabel() {
      if (!this.selectedCommunication) {
        return '未选择通信对'
      }
      return `当前：${this.selectedCommunication.source} → ${this.selectedCommunication.target}`
    },
    pairChartSubtitle() {
      if (!this.selectedCommunication) {
        return ''
      }
      return `${this.selectedCommunication.protocol} | ${this.selectedCommunication.source} → ${this.selectedCommunication.target}`
    }
  },
  watch: {
    analysisGranularity() {
      this.updateTimeSeriesChart()
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.selectedCommunication = this.topCommunications[0] || null
      this.initCharts()
      window.addEventListener('resize', this.handleResize)
      this.$nextTick(() => {
        if (this.$refs.topnTable && this.selectedCommunication) {
          this.$refs.topnTable.setCurrentRow(this.selectedCommunication)
        }
      })
    })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
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
      const baseTimes = this.topCommunications[0]?.timeline.map(item => item.time) || []
      const totals = baseTimes.map((_, idx) =>
        this.topCommunications.reduce((sum, pair) => sum + (pair.timeline[idx]?.value || 0), 0)
      )
      this.charts.overview.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 32, left: 32, right: 16, bottom: 32 },
        xAxis: {
          type: 'category',
          data: baseTimes,
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
      const data = [
        { value: 42, name: 'HTTPS' },
        { value: 28, name: 'HTTP' },
        { value: 18, name: 'MQTT' },
        { value: 8, name: 'IEC104' },
        { value: 4, name: 'Modbus' }
      ]
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
      const categories = this.selectedCommunication?.timeline.map(item => item.time) || []
      const values = this.selectedCommunication?.timeline.map(item => item.value) || []
      let series = []
      if (this.selectedCommunication) {
        series = [
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
      }
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
        series
      })
    },
    initTimeSeriesChart() {
      if (!this.$refs.timeSeriesChart) return
      this.charts.timeSeries = echarts.init(this.$refs.timeSeriesChart)
      this.updateTimeSeriesChart()
    },
    updateTimeSeriesChart() {
      if (!this.charts.timeSeries) return
      const preset = this.timeSeriesPresets[this.analysisGranularity]
      if (!preset) return
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
          name: 'GB',
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
      this.$message.success('筛选条件已应用')
    },
    resetFilters() {
      this.filters = {
        timeRange: [],
        ip: '',
        port: '',
        protocol: ''
      }
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
