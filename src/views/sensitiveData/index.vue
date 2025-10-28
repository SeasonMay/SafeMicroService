<template>
  <div class="sensitive-data-page">
    <el-row :gutter="16" class="stat-row">
      <el-col v-for="card in statCards" :key="card.title" :xs="12" :sm="6" :md="6" :lg="6">
        <el-card shadow="hover" class="stat-card">
          <div class="stat-header">
            <span>{{ card.title }}</span>
            <el-tag :type="card.tagType" size="mini">{{ card.tag }}</el-tag>
          </div>
          <div class="stat-value">{{ card.value }}</div>
          <div class="stat-subtitle">{{ card.subtitle }}</div>
        </el-card>
      </el-col>
    </el-row>

    <el-card shadow="hover" class="filter-card">
      <div class="filter-group">
        <el-input v-model="filters.keyword" placeholder="搜索敏感字段 / 来源" clearable prefix-icon="el-icon-search" />
        <el-select v-model="filters.risk" placeholder="风险等级" clearable style="width: 160px">
          <el-option label="全部" value="all" />
          <el-option label="高危" value="L4" />
          <el-option label="中危" value="L3" />
          <el-option label="低危" value="L2" />
          <el-option label="提示" value="L1" />
        </el-select>
        <el-switch v-model="filters.onlyExceed" active-text="仅显示超权限" />
        <el-button type="primary" icon="el-icon-refresh" @click="refreshData">刷新数据</el-button>
      </div>
    </el-card>

    <div class="data-layout">
      <aside class="classification-panel">
        <el-card shadow="hover" class="tree-card">
          <div slot="header" class="card-header">
            <span>数据分类</span>
          </div>
          <el-tree
            :data="classificationTree"
            :default-expand-all="true"
            highlight-current
            node-key="id"
            @node-click="handleTreeClick"
          />
        </el-card>
      </aside>

      <section class="table-section">
        <el-card shadow="hover" class="table-card">
          <div slot="header" class="card-header">
            <span>敏感数据列表</span>
            <el-button type="text" size="mini" icon="el-icon-download">导出报表</el-button>
          </div>
          <el-table
            :data="filteredTableData"
            border
            highlight-current-row
            height="420"
            @current-change="handleRowSelect"
          >
            <el-table-column type="index" width="60" label="#" />
            <el-table-column prop="name" label="字段名称" min-width="160" />
            <el-table-column prop="category" label="数据类型" width="140" />
            <el-table-column prop="sensitivity" label="敏感等级" width="120">
              <template slot-scope="scope">
                <el-tag :type="sensitivityTag(scope.row.sensitivity)" size="mini">{{ scope.row.sensitivity }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="sourcePort" label="来源端口" width="140" />
            <el-table-column prop="requestPath" label="请求路径" min-width="200" show-overflow-tooltip />
            <el-table-column prop="srcIp" label="源 IP" width="150" />
            <el-table-column prop="dstIp" label="目的 IP" width="150" />
            <el-table-column prop="permission" label="权限校验" width="140">
              <template slot-scope="scope">
                <el-tag :type="scope.row.permission === '越权访问' ? 'danger' : 'success'" size="mini">{{ scope.row.permission }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="lastSeen" label="最近检测" width="170" />
          </el-table>
        </el-card>
      </section>

      <aside class="detail-panel">
        <el-card shadow="hover" class="detail-card">
          <div slot="header" class="card-header">
            <span>详细信息</span>
          </div>
          <div v-if="selectedRecord" class="detail-content">
            <h3 class="detail-title">{{ selectedRecord.name }}</h3>
            <div class="detail-tags">
              <el-tag :type="sensitivityTag(selectedRecord.sensitivity)" effect="dark" size="mini">{{ selectedRecord.sensitivity }}</el-tag>
              <el-tag :type="selectedRecord.permission === '越权访问' ? 'danger' : 'success'" size="mini">{{ selectedRecord.permission }}</el-tag>
            </div>
            <el-descriptions :column="1" border size="small">
              <el-descriptions-item label="数据类型">{{ selectedRecord.category }}</el-descriptions-item>
              <el-descriptions-item label="来源服务">{{ selectedRecord.service }}</el-descriptions-item>
              <el-descriptions-item label="来源端口">{{ selectedRecord.sourcePort }}</el-descriptions-item>
              <el-descriptions-item label="请求路径">{{ selectedRecord.requestPath }}</el-descriptions-item>
              <el-descriptions-item label="源 → 目的 IP">{{ `${selectedRecord.srcIp} → ${selectedRecord.dstIp}` }}</el-descriptions-item>
              <el-descriptions-item label="检测时间">{{ selectedRecord.lastSeen }}</el-descriptions-item>
              <el-descriptions-item label="风险评估">{{ selectedRecord.riskNote }}</el-descriptions-item>
              <el-descriptions-item label="传输路径">{{ selectedRecord.path.join(' → ') }}</el-descriptions-item>
            </el-descriptions>
          </div>
          <div v-else class="detail-placeholder">
            <el-empty description="选择一条记录查看详情" />
          </div>
        </el-card>
      </aside>
    </div>

    <el-card shadow="hover" class="lineage-card">
      <div slot="header" class="card-header">
        <span>敏感数据血缘关系</span>
        <el-tag type="info" size="mini">{{ lineageLabel }}</el-tag>
      </div>
      <div ref="lineageChart" class="lineage-canvas" />
    </el-card>
  </div>
</template>

<script>
import * as echarts from 'echarts'

export default {
  name: 'SensitiveData',
  data() {
    return {
      statCards: [
        { title: '高敏字段告警', value: '312', subtitle: '本日新增 24 条', tag: '调控域', tagType: 'danger' },
        { title: '策略类敏感字段', value: '68', subtitle: '关注调度策略增量', tag: '策略防护', tagType: 'warning' },
        { title: '运行监测关键字段', value: '146', subtitle: '覆盖主站与边缘节点', tag: '运行域', tagType: 'info' },
        { title: '市场结算敏感字段', value: '57', subtitle: '结算口径波动 3%', tag: '交易域', tagType: 'success' }
      ],
      filters: {
        keyword: '',
        risk: 'all',
        onlyExceed: false
      },
      classificationTree: [
        {
          id: 1,
          label: '调度控制域',
          children: [
            { id: 11, label: '安全调度指令 (L4)' },
            { id: 12, label: '运行约束参数 (L3)' },
            { id: 13, label: '潮流计算断面 (L3)' }
          ]
        },
        {
          id: 2,
          label: '运行监测域',
          children: [
            { id: 21, label: '实时遥信 (L2)' },
            { id: 22, label: '遥测基线 (L3)' }
          ]
        },
        {
          id: 3,
          label: '市场交易域',
          children: [
            { id: 31, label: '购售电合同 (L4)' },
            { id: 32, label: '出清凭证 (L3)' }
          ]
        },
        {
          id: 4,
          label: '安全支撑域',
          children: [
            { id: 41, label: '策略发布稿 (L3)' },
            { id: 42, label: '加密密钥片段 (L4)' }
          ]
        }
      ],
      tableData: [
        {
          id: 1,
          name: 'dispatch_master_plan',
          category: '调度控制域',
          sensitivity: 'L4',
          sourcePort: 'TCP:7010',
          service: 'dispatch-strategy-center',
          permission: '越权访问',
          lastSeen: '2024-08-12 09:56',
          riskNote: '边缘协调器尝试拉取主站调度方案，疑似越权同步。',
          path: ['dispatch-strategy-center', 'integration-bus', 'edge-coordinator'],
          srcIp: '10.20.18.45',
          dstIp: '172.16.10.3',
          requestPath: '/v1/strategy/master-plan/export'
        },
        {
          id: 2,
          name: 'scada_limit_profile',
          category: '运行监测域',
          sensitivity: 'L3',
          sourcePort: 'TCP:6080',
          service: 'scada-core',
          permission: '受控访问',
          lastSeen: '2024-08-12 09:48',
          riskNote: '遥测阈值曲线按计划推送，处于受控白名单。',
          path: ['scada-core', 'data-hub', 'analysis-engine'],
          srcIp: '10.30.5.18',
          dstIp: '172.16.21.9',
          requestPath: '/telemetry/limit-profile/publish'
        },
        {
          id: 3,
          name: 'market_clearing_ticket',
          category: '市场交易域',
          sensitivity: 'L4',
          sourcePort: 'HTTPS:9443',
          service: 'market-clearing-service',
          permission: '越权访问',
          lastSeen: '2024-08-12 09:37',
          riskNote: '检测到边缘结算节点批量导出结算凭证，需核实操作来源。',
          path: ['market-clearing-service', 'settlement-adapter', 'external-clearing-node'],
          srcIp: '10.50.2.62',
          dstIp: '172.16.40.27',
          requestPath: '/settlement/v2/ticket/download'
        },
        {
          id: 4,
          name: 'grid_security_token',
          category: '安全支撑域',
          sensitivity: 'L4',
          sourcePort: 'TCP:5520',
          service: 'crypto-vault',
          permission: '越权访问',
          lastSeen: '2024-08-12 09:18',
          riskNote: '授权存储库外出现密钥片段调用，请立即核查密钥分发链路。',
          path: ['crypto-vault', 'secure-gateway', 'maintenance-console'],
          srcIp: '10.10.2.5',
          dstIp: '172.16.5.44',
          requestPath: '/token/segment/fetch'
        },
        {
          id: 5,
          name: 'flexible_load_order',
          category: '调度控制域',
          sensitivity: 'L2',
          sourcePort: 'MQTT:1883',
          service: 'demand-response-hub',
          permission: '受控访问',
          lastSeen: '2024-08-12 09:05',
          riskNote: '柔性负荷调度令按策略分发，已验证边缘节点响应。',
          path: ['demand-response-hub', 'event-bus', 'flex-load-agent'],
          srcIp: '10.25.11.33',
          dstIp: '172.16.18.56',
          requestPath: '/orders/flex-load/dispatch'
        }
      ],
      selectedRecord: null,
      lineageChart: null
    }
  },
  computed: {
    filteredTableData() {
      return this.tableData.filter(item => {
        if (this.filters.keyword && !`${item.name}${item.category}${item.service}${item.requestPath}${item.srcIp}${item.dstIp}`.includes(this.filters.keyword)) return false
        if (this.filters.risk && this.filters.risk !== 'all' && item.sensitivity !== this.filters.risk) return false
        if (this.filters.onlyExceed && item.permission !== '越权访问') return false
        return true
      })
    },
    lineageLabel() {
      return this.selectedRecord ? `${this.selectedRecord.name} 血缘追踪` : '请选择一条敏感数据记录'
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.initLineageChart()
      if (this.tableData.length) {
        this.handleRowSelect(this.tableData[0])
      }
    })
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.handleResize)
    if (this.lineageChart) {
      this.lineageChart.dispose()
      this.lineageChart = null
    }
  },
  methods: {
    sensitivityTag(level) {
      switch (level) {
        case 'L4':
          return 'danger'
        case 'L3':
          return 'warning'
        case 'L2':
          return 'info'
        default:
          return 'success'
      }
    },
    refreshData() {
      this.$message.success('敏感数据视图已刷新')
    },
    handleTreeClick(node) {
      if (node.label) {
        const label = node.label.split(' ')[0]
        this.filters.keyword = label
      }
    },
    handleRowSelect(row) {
      this.selectedRecord = row || null
      this.renderLineage()
    },
    initLineageChart() {
      if (!this.$refs.lineageChart) return
      this.lineageChart = echarts.init(this.$refs.lineageChart)
      this.renderLineage()
      window.addEventListener('resize', this.handleResize)
    },
    renderLineage() {
      if (!this.lineageChart) return
      const record = this.selectedRecord
      const nodes = record
        ? record.path.map((name, index) => ({
          id: `${name}-${index}`,
          name,
          symbolSize: index === 0 ? 58 : index === record.path.length - 1 ? 52 : 46,
          itemStyle: { color: index === record.path.length - 1 ? '#F56C6C' : '#409EFF' }
        }))
        : []
      const links = []
      if (record) {
        for (let i = 0; i < record.path.length - 1; i++) {
          links.push({
            source: `${record.path[i]}-${i}`,
            target: `${record.path[i + 1]}-${i + 1}`,
            lineStyle: { color: i === record.path.length - 2 ? '#F56C6C' : '#67C23A', width: 2 }
          })
        }
      }
      this.lineageChart.setOption({
        tooltip: {
          trigger: 'item'
        },
        series: [
          {
            type: 'graph',
            layout: 'force',
            roam: true,
            draggable: true,
            data: nodes,
            links,
            label: { show: true },
            force: { repulsion: 200, edgeLength: 140 }
          }
        ]
      })
    },
    handleResize() {
      if (this.lineageChart) {
        this.lineageChart.resize()
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.sensitive-data-page {
  padding: 16px;
  background: #f5f7fa;
  min-height: 100%;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.stat-row {
  margin-bottom: 0;
}

.stat-card {
  border-radius: 12px;
  .stat-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    color: #606266;
    margin-bottom: 8px;
  }
  .stat-value {
    font-size: 30px;
    font-weight: 700;
    color: #303133;
    margin-bottom: 6px;
  }
  .stat-subtitle {
    font-size: 13px;
    color: #909399;
  }
}

.filter-card {
  border-radius: 12px;
  .filter-group {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
  }
}

.data-layout {
  display: grid;
  grid-template-columns: 260px 1fr 320px;
  gap: 16px;
}

.tree-card,
.detail-card,
.table-card {
  border-radius: 12px;
}

.table-section {
  width: 100%;
}

.detail-panel {
  display: flex;
}

.detail-content {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.detail-title {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
}

.detail-tags {
  display: flex;
  gap: 8px;
}

.detail-placeholder {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.lineage-card {
  border-radius: 12px;
}

.lineage-canvas {
  width: 100%;
  height: 320px;
}

@media (max-width: 1280px) {
  .data-layout {
    grid-template-columns: 220px 1fr;
    grid-template-areas:
      'tree table'
      'tree detail';
  }
  .detail-panel {
    grid-column: span 2;
  }
}

@media (max-width: 1024px) {
  .data-layout {
    grid-template-columns: 1fr;
  }
  .detail-panel {
    order: 3;
  }
  .lineage-canvas {
    height: 280px;
  }
}
</style>
