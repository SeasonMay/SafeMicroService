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
        <el-radio-group v-model="filters.viewMode" size="small">
          <el-radio-button v-for="item in viewModes" :key="item.value" :label="item.value">{{ item.label }}</el-radio-button>
        </el-radio-group>
        <el-button type="primary" icon="el-icon-search" @click="applyFilters">应用筛选</el-button>
        <el-button icon="el-icon-refresh" @click="resetFilters">重置</el-button>
      </div>
    </el-card>

    <el-tabs v-model="activeTab" class="monitor-tabs" @tab-click="handleTabChange">
      <el-tab-pane label="实时监控" name="realtime">
        <el-row :gutter="16">
          <el-col :xs="24" :md="16">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>实时流量波形图</span>
                <el-tag type="success" size="mini">实时刷新</el-tag>
              </div>
              <div ref="realtimeChart" class="chart-area" />
            </el-card>
          </el-col>
          <el-col :xs="24" :md="8">
            <el-card shadow="hover">
              <div slot="header" class="card-header">
                <span>协议分布</span>
                <el-tag type="info" size="mini">当前 5 分钟</el-tag>
              </div>
              <div ref="protocolChart" class="chart-area small" />
            </el-card>
          </el-col>
        </el-row>
        <el-card shadow="hover" class="table-card">
          <div slot="header" class="card-header">
            <span>TOP N 通信对</span>
            <el-switch v-model="autoRefresh" active-text="自动刷新" />
          </div>
          <el-table :data="topCommunications" border height="240">
            <el-table-column prop="source" label="源地址" min-width="150" />
            <el-table-column prop="target" label="目标地址" min-width="150" />
            <el-table-column prop="protocol" label="协议" width="100">
              <template slot-scope="scope">
                <el-tag :type="protocolTag(scope.row.protocol)" size="mini">{{ scope.row.protocol }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="pps" label="包率 (pps)" width="140" />
            <el-table-column prop="bandwidth" label="带宽 (Mbps)" width="160" />
            <el-table-column prop="trend" label="趋势" width="100">
              <template slot-scope="scope">
                <span :class="['trend', scope.row.trend >= 0 ? 'up' : 'down']">
                  {{ scope.row.trend >= 0 ? '+' : '' }}{{ scope.row.trend }}%
                </span>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-tab-pane>

      <el-tab-pane label="流量分析" name="analysis">
        <el-row :gutter="16">
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
                <span>协议占比分析</span>
                <el-tag type="warning" size="mini">历史区间</el-tag>
              </div>
              <div ref="stackedChart" class="chart-area" />
            </el-card>
          </el-col>
        </el-row>
        <el-card shadow="hover" class="table-card">
          <div slot="header" class="card-header">
            <span>地理分布分析</span>
            <el-button type="text" size="mini">导出数据</el-button>
          </div>
          <el-table :data="geoDistribution" border height="260">
            <el-table-column prop="region" label="地区" min-width="150" />
            <el-table-column prop="traffic" label="流量 (GB)" width="140" />
            <el-table-column prop="ratio" label="占比" width="120">
              <template slot-scope="scope">
                <el-progress :percentage="scope.row.ratio" :stroke-width="6" />
              </template>
            </el-table-column>
            <el-table-column prop="trend" label="环比" width="120">
              <template slot-scope="scope">
                <span :class="['trend', scope.row.trend >= 0 ? 'up' : 'down']">
                  {{ scope.row.trend >= 0 ? '+' : '' }}{{ scope.row.trend }}%
                </span>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
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
        protocol: '',
        viewMode: 'realtime'
      },
      viewModes: [
        { label: '实时', value: 'realtime' },
        { label: '历史', value: 'historical' }
      ],
      protocols: ['HTTP', 'HTTPS', 'MQTT', 'IEC104', 'Modbus'],
      activeTab: 'realtime',
      autoRefresh: true,
      analysisGranularity: 'hour',
      topCommunications: [
        { source: '10.1.2.11', target: '172.16.0.21', protocol: 'HTTPS', pps: 4210, bandwidth: 86, trend: 12 },
        { source: '10.1.2.14', target: '172.16.0.34', protocol: 'HTTP', pps: 3560, bandwidth: 42, trend: -5 },
        { source: '10.1.3.20', target: '172.16.0.82', protocol: 'MQTT', pps: 2112, bandwidth: 33, trend: 8 },
        { source: '10.2.1.51', target: '172.16.0.99', protocol: 'IEC104', pps: 1920, bandwidth: 28, trend: 4 },
        { source: '10.2.5.18', target: '172.16.0.45', protocol: 'Modbus', pps: 1422, bandwidth: 21, trend: -2 }
      ],
      geoDistribution: [
        { region: '华东调度中心', traffic: 162, ratio: 42, trend: 6 },
        { region: '华北监控中心', traffic: 128, ratio: 33, trend: -4 },
        { region: '华南数据节点', traffic: 78, ratio: 18, trend: 3 },
        { region: '西部边缘节点', traffic: 24, ratio: 7, trend: 1 }
      ],
      anomalyEvents: [
        { time: '2024-07-18 13:40', type: 'DDoS 攻击', tag: 'danger', source: '北部探针 #3', description: '检测到大量 SYN 洪水攻击，已启用丢弃策略', status: '已阻断', statusTag: 'success' },
        { time: '2024-07-18 12:58', type: '端口扫描', tag: 'warning', source: '南部探针 #1', description: '内外网边界发现高频端口探测行为', status: '观察中', statusTag: 'warning' },
        { time: '2024-07-18 11:35', type: '异常大流量', tag: 'info', source: '东部探针 #2', description: '归档服务上传数据超出阈值，已通知负责人', status: '待确认', statusTag: 'info' }
      ],
      mitigationActions: [
        { id: 1, title: '触发自动限速策略', description: '对攻击源 10.1.2.254 启动 5 分钟带宽限制', time: '13:42', type: 'danger' },
        { id: 2, title: '联动防火墙封禁', description: '推送 6 条封禁规则至边界防火墙', time: '13:44', type: 'warning' },
        { id: 3, title: '通知服务负责人', description: '消息队列服务负责人已确认异常波动', time: '12:02', type: 'success' }
      ],
      charts: {
        realtime: null,
        protocol: null,
        timeSeries: null,
        stacked: null,
        anomaly: null
      }
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.initCharts()
      window.addEventListener('resize', this.handleResize)
    })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
    this.disposeCharts()
  },
  methods: {
    initCharts() {
      this.initRealtimeChart()
      this.initProtocolChart()
      this.initTimeSeriesChart()
      this.initStackedChart()
      this.initAnomalyChart()
    },
    initRealtimeChart() {
      if (!this.$refs.realtimeChart) return
      this.charts.realtime = echarts.init(this.$refs.realtimeChart)
      const base = Array.from({ length: 20 }).map((_, idx) => ({
        time: `${idx + 1}秒`,
        value: Math.round(30 + Math.random() * 20)
      }))
      this.charts.realtime.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 32, left: 32, right: 16, bottom: 32 },
        xAxis: {
          type: 'category',
          data: base.map(item => item.time),
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
            name: '实时流量',
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
            data: base.map(item => item.value)
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
    initTimeSeriesChart() {
      if (!this.$refs.timeSeriesChart) return
      this.charts.timeSeries = echarts.init(this.$refs.timeSeriesChart)
      const categories = ['00:00', '04:00', '08:00', '12:00', '16:00', '20:00']
      const seriesData = [320, 450, 560, 620, 580, 490]
      this.charts.timeSeries.setOption({
        tooltip: { trigger: 'axis' },
        grid: { top: 32, left: 36, right: 16, bottom: 32 },
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
            name: '总流量',
            type: 'line',
            smooth: true,
            symbol: 'circle',
            itemStyle: { color: '#67C23A' },
            lineStyle: { color: '#67C23A', width: 3 },
            data: seriesData
          }
        ]
      })
    },
    initStackedChart() {
      if (!this.$refs.stackedChart) return
      this.charts.stacked = echarts.init(this.$refs.stackedChart)
      const categories = ['周一', '周二', '周三', '周四', '周五', '周六', '周日']
      this.charts.stacked.setOption({
        tooltip: { trigger: 'axis' },
        legend: { bottom: 0 },
        grid: { top: 32, left: 32, right: 16, bottom: 48 },
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
            name: 'HTTPS',
            type: 'bar',
            stack: 'total',
            emphasis: { focus: 'series' },
            data: [120, 132, 101, 134, 90, 230, 210]
          },
          {
            name: 'HTTP',
            type: 'bar',
            stack: 'total',
            emphasis: { focus: 'series' },
            data: [220, 182, 191, 234, 290, 330, 310]
          },
          {
            name: 'MQTT',
            type: 'bar',
            stack: 'total',
            emphasis: { focus: 'series' },
            data: [150, 232, 201, 154, 190, 330, 410]
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
        protocol: '',
        viewMode: 'realtime'
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
}

.table-card {
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
}
</style>
