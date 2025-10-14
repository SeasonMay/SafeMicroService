<template>
  <div class="overview-dashboard">
    <el-row :gutter="16" class="kpi-row">
      <el-col v-for="item in kpiCards" :key="item.key" :xs="12" :sm="12" :md="6" :lg="6">
        <el-card shadow="hover" class="kpi-card">
          <div class="kpi-header">
            <span class="kpi-title">{{ item.title }}</span>
            <el-tag :type="item.trend > 0 ? 'danger' : 'success'" size="mini">
              {{ item.trend > 0 ? '+' : '' }}{{ item.trend }}%
            </el-tag>
          </div>
          <div class="kpi-value">{{ item.value }}</div>
          <div class="kpi-footer">{{ item.subtitle }}</div>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="16" class="content-row">
      <el-col :xs="24" :sm="24" :md="16">
        <el-card shadow="hover" class="security-posture-card">
          <div slot="header" class="card-header">
            <span>安全态势总览</span>
            <el-tag type="info" effect="dark" size="mini">实时评估</el-tag>
          </div>
          <div class="posture-content">
            <div ref="gaugeChart" class="gauge-chart" />
            <div class="quadrant-panel">
              <div v-for="quadrant in quadrants" :key="quadrant.key" class="quadrant-card">
                <div class="quadrant-title">
                  <i :class="quadrant.icon" />
                  <span>{{ quadrant.title }}</span>
                </div>
                <div class="quadrant-metrics">
                  <span class="metric-value">{{ quadrant.metric }}</span>
                  <span class="metric-label">{{ quadrant.label }}</span>
                </div>
                <el-progress :percentage="quadrant.percentage" :stroke-width="8" :color="quadrant.color" />
                <div class="quadrant-detail">
                  <span v-for="tag in quadrant.tags" :key="tag" class="detail-tag">{{ tag }}</span>
                </div>
              </div>
            </div>
          </div>
        </el-card>
      </el-col>
      <el-col :xs="24" :sm="24" :md="8">
        <el-card shadow="hover" class="event-card">
          <div slot="header" class="card-header">
            <span>最新安全事件</span>
            <el-button type="text" size="mini" @click="acknowledgeAll">全部确认</el-button>
          </div>
          <el-timeline class="event-timeline">
            <el-timeline-item
              v-for="event in securityEvents"
              :key="event.id"
              :type="event.type"
              :timestamp="event.time"
              :hollow="event.type === 'info'"
            >
              <div class="event-item">
                <div class="event-title">{{ event.title }}</div>
                <div class="event-meta">
                  <el-tag :type="event.tagType" size="mini">{{ event.severity }}</el-tag>
                  <span class="event-source">{{ event.source }}</span>
                </div>
                <p class="event-description">{{ event.description }}</p>
              </div>
            </el-timeline-item>
          </el-timeline>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="16" class="bottom-row">
      <el-col :xs="24" :sm="24" :md="16">
        <el-card shadow="hover" class="control-card">
          <div slot="header" class="card-header">
            <span>采集控制策略下发</span>
            <el-tag :type="collectControl.running ? 'success' : 'info'" effect="plain" size="mini">
              {{ collectControl.running ? '采集中' : '已停止' }}
            </el-tag>
          </div>
          <el-form :model="collectControl" label-position="top" class="control-form">
            <el-row :gutter="20">
              <el-col :sm="12" :xs="24">
                <el-form-item label="采集开关">
                  <el-switch
                    v-model="collectControl.running"
                    active-text="开启"
                    inactive-text="关闭"
                    @change="toggleCollect"
                  />
                </el-form-item>
              </el-col>
              <el-col :sm="12" :xs="24">
                <el-form-item label="采集模式">
                  <el-radio-group v-model="collectControl.mode" size="small">
                    <el-radio-button label="incremental">增量</el-radio-button>
                    <el-radio-button label="full">全量</el-radio-button>
                  </el-radio-group>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :sm="12" :xs="24">
                <el-form-item label="采样率">
                  <el-slider v-model="collectControl.sampleRate" :step="5" show-stops show-input />
                </el-form-item>
              </el-col>
              <el-col :sm="12" :xs="24">
                <el-form-item label="BPF / 策略规则">
                  <el-input
                    v-model="collectControl.bpf"
                    type="textarea"
                    :rows="3"
                    placeholder="tcp port 443 and not host 10.10.1.23"
                  />
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :sm="8" :xs="24">
                <el-form-item label="节点在线">
                  <div class="inline-stat">{{ collectControl.nodesOnline }} 台</div>
                </el-form-item>
              </el-col>
              <el-col :sm="8" :xs="24">
                <el-form-item label="丢包率">
                  <el-progress :percentage="collectControl.dropRate" status="exception" />
                </el-form-item>
              </el-col>
              <el-col :sm="8" :xs="24">
                <el-form-item label="背压">
                  <el-tag type="warning">{{ collectControl.backPressure }}</el-tag>
                </el-form-item>
              </el-col>
            </el-row>
            <el-row :gutter="20">
              <el-col :sm="12" :xs="24">
                <el-form-item label="最近 ACK">
                  <el-input v-model="collectControl.lastAck" readonly />
                </el-form-item>
              </el-col>
              <el-col :sm="12" :xs="24">
                <el-form-item label="最近 ERR">
                  <el-input v-model="collectControl.lastErr" readonly />
                </el-form-item>
              </el-col>
            </el-row>
            <div class="control-actions">
              <el-button type="primary" icon="el-icon-position">下发策略</el-button>
              <el-button type="success" icon="el-icon-refresh">刷新节点</el-button>
              <el-button type="warning" icon="el-icon-back">一键回滚</el-button>
            </div>
          </el-form>
        </el-card>
      </el-col>
      <el-col :xs="24" :sm="24" :md="8">
        <el-card shadow="hover" class="threat-summary-card">
          <div slot="header" class="card-header">
            <span>威胁热度排行</span>
            <el-tag type="danger" size="mini" effect="dark">TOP 5</el-tag>
          </div>
          <el-table :data="threatHotspots" size="mini" border height="280">
            <el-table-column prop="name" label="威胁" min-width="120" />
            <el-table-column prop="trend" label="趋势" width="90">
              <template slot-scope="scope">
                <span :class="['trend', scope.row.trend >= 0 ? 'up' : 'down']">
                  {{ scope.row.trend >= 0 ? '+' : '' }}{{ scope.row.trend }}%
                </span>
              </template>
            </el-table-column>
            <el-table-column prop="score" label="风险评分" width="110">
              <template slot-scope="scope">
                <el-progress :percentage="scope.row.score" :color="scope.row.color" :stroke-width="6" />
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import * as echarts from 'echarts'

export default {
  name: 'Dashboard',
  data() {
    return {
      kpiCards: [
        { key: 'traffic', title: '实时流量吞吐量', subtitle: 'Gbps', value: '12.6', trend: 6.8 },
        { key: 'services', title: '活跃服务数量', subtitle: '在线服务', value: '148', trend: 2.1 },
        { key: 'threats', title: '检测威胁数量', subtitle: '近 24 小时', value: '37', trend: -12.5 },
        { key: 'risk', title: '敏感数据风险等级', subtitle: '全网平均', value: '中 (2.6)', trend: -4.3 }
      ],
      quadrants: [
        {
          key: 'hidden',
          title: '隐藏接口发现',
          label: '新增 8 个',
          metric: '34',
          percentage: 68,
          color: '#409EFF',
          icon: 'el-icon-view',
          tags: ['高危 6', '中危 18', '低危 10']
        },
        {
          key: 'abuse',
          title: '接口滥用',
          label: '异常调用模式',
          metric: '19',
          percentage: 52,
          color: '#E6A23C',
          icon: 'el-icon-warning-outline',
          tags: ['越权 5', '跨域 3', '暴力请求 11']
        },
        {
          key: 'traffic',
          title: '接口流量异常',
          label: '突增/扫描/长连',
          metric: '27',
          percentage: 74,
          color: '#F56C6C',
          icon: 'el-icon-data-analysis',
          tags: ['突增 9', '扫描 12', '长连接 6']
        },
        {
          key: 'privacy',
          title: '隐私数据泄露',
          label: 'L1~L4 风险分布',
          metric: '11',
          percentage: 36,
          color: '#67C23A',
          icon: 'el-icon-lock',
          tags: ['L4:2', 'L3:4', 'L2:3', 'L1:2']
        }
      ],
      collectControl: {
        running: true,
        mode: 'incremental',
        sampleRate: 65,
        bpf: '',
        nodesOnline: 42,
        dropRate: 8,
        backPressure: '轻微',
        lastAck: '2024-07-18 13:22:09',
        lastErr: '无'
      },
      securityEvents: [
        {
          id: 1,
          title: '发现未授权微服务访问敏感接口',
          severity: '高危',
          source: '服务网关 #3',
          tagType: 'danger',
          type: 'danger',
          time: '13:45',
          description: 'API /energy/dispatch 被测试环境实例访问，已阻断。'
        },
        {
          id: 2,
          title: '跨区域数据同步出现异常流量',
          severity: '中危',
          source: '流量探针-北部',
          tagType: 'warning',
          type: 'warning',
          time: '13:20',
          description: '检测到疑似端口扫描行为，触发速率限制策略。'
        },
        {
          id: 3,
          title: '白名单微服务实例新增',
          severity: '提示',
          source: '服务注册中心',
          tagType: 'info',
          type: 'info',
          time: '12:58',
          description: '微服务 storage-adapter-07 加入白名单列表。'
        },
        {
          id: 4,
          title: '敏感数据外发预警解除',
          severity: '已处理',
          source: '敏感数据守护',
          tagType: 'success',
          type: 'success',
          time: '12:36',
          description: '外发数据策略校验通过，风险等级降至 L1。'
        }
      ],
      threatHotspots: [
        { name: '异常流量突增', trend: 12, score: 78, color: '#F56C6C' },
        { name: '接口暴力猜测', trend: 8, score: 65, color: '#E6A23C' },
        { name: '未授权调用', trend: -5, score: 54, color: '#909399' },
        { name: '敏感数据外泄', trend: 4, score: 61, color: '#67C23A' },
        { name: '端口扫描', trend: 2, score: 49, color: '#409EFF' }
      ],
      gaugeChart: null,
      securityScore: 82
    }
  },
  watch: {
    securityScore() {
      this.updateGauge()
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.initGauge()
      window.addEventListener('resize', this.handleResize)
    })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
    this.disposeCharts()
  },
  methods: {
    initGauge() {
      if (!this.$refs.gaugeChart) return
      this.gaugeChart = echarts.init(this.$refs.gaugeChart)
      this.updateGauge()
    },
    updateGauge() {
      if (!this.gaugeChart) return
      const gradient = new echarts.graphic.LinearGradient(0, 0, 1, 1, [
        { offset: 0, color: '#67C23A' },
        { offset: 0.5, color: '#E6A23C' },
        { offset: 1, color: '#F56C6C' }
      ])
      this.gaugeChart.setOption({
        series: [
          {
            type: 'gauge',
            startAngle: 210,
            endAngle: -30,
            min: 0,
            max: 100,
            splitNumber: 10,
            radius: '90%',
            axisLine: {
              lineStyle: {
                width: 18,
                color: [[0.6, '#67C23A'], [0.8, '#E6A23C'], [1, '#F56C6C']]
              }
            },
            progress: {
              show: true,
              width: 18,
              itemStyle: {
                color: gradient,
                shadowColor: 'rgba(0, 0, 0, 0.2)',
                shadowBlur: 6
              }
            },
            pointer: {
              length: '65%',
              width: 4
            },
            axisTick: {
              distance: -24,
              length: 4,
              lineStyle: { color: '#909399' }
            },
            splitLine: {
              distance: -26,
              length: 10,
              lineStyle: { color: '#909399' }
            },
            axisLabel: {
              color: '#909399',
              distance: -40,
              fontSize: 12
            },
            detail: {
              valueAnimation: true,
              formatter: '{value} 分',
              fontSize: 24,
              color: '#303133',
              offsetCenter: [0, '60%']
            },
            title: {
              offsetCenter: [0, '-60%'],
              fontSize: 16,
              color: '#606266'
            },
            data: [
              {
                value: this.securityScore,
                name: '安全健康评分'
              }
            ]
          }
        ]
      })
    },
    handleResize() {
      if (this.gaugeChart) {
        this.gaugeChart.resize()
      }
    },
    disposeCharts() {
      if (this.gaugeChart) {
        this.gaugeChart.dispose()
        this.gaugeChart = null
      }
    },
    toggleCollect(value) {
      this.$message({
        type: value ? 'success' : 'info',
        message: value ? '采集已开启' : '采集已停止'
      })
    },
    acknowledgeAll() {
      this.$message.success('全部事件已确认')
    }
  }
}
</script>

<style lang="scss" scoped>
.overview-dashboard {
  padding: 16px;
  background: var(--dashboard-bg, #f5f7fa);
  min-height: 100%;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 600;
  color: #303133;
}

.kpi-row {
  margin-bottom: 16px;
}

.kpi-card {
  border-radius: 12px;
  overflow: hidden;
  .kpi-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;
    color: #606266;
  }
  .kpi-title {
    font-size: 14px;
  }
  .kpi-value {
    font-size: 32px;
    font-weight: 700;
    color: #303133;
    margin-bottom: 8px;
  }
  .kpi-footer {
    color: #909399;
    font-size: 13px;
  }
}

.content-row {
  margin-bottom: 16px;
}

.security-posture-card {
  height: 100%;
  .posture-content {
    display: flex;
    flex-wrap: wrap;
    gap: 24px;
  }
}

.gauge-chart {
  flex: 0 0 320px;
  height: 280px;
}

.quadrant-panel {
  display: grid;
  gap: 16px;
  flex: 1;
  grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
}

.quadrant-card {
  background: linear-gradient(160deg, rgba(64, 158, 255, 0.08), rgba(255, 255, 255, 0.92));
  padding: 16px;
  border-radius: 12px;
  box-shadow: inset 0 0 0 1px rgba(64, 158, 255, 0.12);
  display: flex;
  flex-direction: column;
  gap: 8px;
  .quadrant-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-weight: 600;
    color: #303133;
    i {
      color: #409eff;
    }
  }
  .quadrant-metrics {
    display: flex;
    flex-direction: column;
    .metric-value {
      font-size: 24px;
      font-weight: 700;
      color: #303133;
      line-height: 1.1;
    }
    .metric-label {
      font-size: 12px;
      color: #909399;
    }
  }
  .quadrant-detail {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
  }
  .detail-tag {
    font-size: 12px;
    padding: 2px 8px;
    background: rgba(144, 147, 153, 0.12);
    border-radius: 10px;
    color: #606266;
  }
}

.event-card {
  .event-timeline {
    max-height: 420px;
    overflow-y: auto;
  }
  .event-item {
    .event-title {
      font-weight: 600;
      margin-bottom: 4px;
    }
    .event-meta {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 12px;
      color: #909399;
      margin-bottom: 6px;
    }
    .event-description {
      font-size: 13px;
      color: #606266;
      margin: 0;
    }
  }
}

.control-card {
  .control-form {
    .control-actions {
      display: flex;
      gap: 12px;
      margin-top: 12px;
    }
    .inline-stat {
      font-size: 24px;
      font-weight: 700;
      color: #303133;
    }
  }
}

.threat-summary-card {
  .trend {
    font-weight: 600;
    &.up {
      color: #f56c6c;
    }
    &.down {
      color: #67c23a;
    }
  }
}

@media (max-width: 1024px) {
  .posture-content {
    flex-direction: column;
  }
  .gauge-chart {
    width: 100%;
    flex: 1 1 auto;
  }
}
</style>
