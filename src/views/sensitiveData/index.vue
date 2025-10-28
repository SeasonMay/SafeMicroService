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
        <el-select v-model="filters.compliance" placeholder="合规状态" clearable style="width: 160px">
          <el-option label="全部" value="all" />
          <el-option label="已合规" value="ok" />
          <el-option label="待整改" value="todo" />
          <el-option label="违规" value="violation" />
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
            <el-table-column prop="source" label="来源端口" width="140" />
            <el-table-column prop="permission" label="权限级别" width="140">
              <template slot-scope="scope">
                <el-tag :type="scope.row.permission === '合规' ? 'success' : 'warning'" size="mini">{{ scope.row.permission }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="status" label="合规状态" width="140">
              <template slot-scope="scope">
                <el-tag :type="statusTag(scope.row.status)" size="mini">{{ scope.row.statusLabel }}</el-tag>
              </template>
            </el-table-column>
            <el-table-column prop="lastSeen" label="最近检测" width="160" />
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
              <el-tag :type="selectedRecord.permission === '合规' ? 'success' : 'warning'" size="mini">{{ selectedRecord.permission }}</el-tag>
            </div>
            <el-descriptions :column="1" border size="small">
              <el-descriptions-item label="数据类型">{{ selectedRecord.category }}</el-descriptions-item>
              <el-descriptions-item label="来源服务">{{ selectedRecord.service }}</el-descriptions-item>
              <el-descriptions-item label="来源端口">{{ selectedRecord.source }}</el-descriptions-item>
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
        { title: '敏感数据总量', value: '1,268', subtitle: '本周新增 128 条', tag: 'L2-L4', tagType: 'danger' },
        { title: '分类覆盖率', value: '92%', subtitle: '共 18 类数据资产', tag: '分类完整', tagType: 'success' },
        { title: '风险等级分布', value: '高危 12 / 中危 46', subtitle: '风险趋势下降 8%', tag: '风险监控', tagType: 'warning' },
        { title: '合规状态', value: '78%', subtitle: '待整改 9 项', tag: '合规巡检', tagType: 'info' }
      ],
      filters: {
        keyword: '',
        risk: 'all',
        compliance: 'all',
        onlyExceed: false
      },
      classificationTree: [
        {
          id: 1,
          label: '个人信息',
          children: [
            { id: 11, label: '基础信息 (L2)' },
            { id: 12, label: '身份认证 (L3)' },
            { id: 13, label: '金融账户 (L4)' }
          ]
        },
        {
          id: 2,
          label: '企业机密',
          children: [
            { id: 21, label: '运营数据 (L3)' },
            { id: 22, label: '调度策略 (L4)' }
          ]
        },
        {
          id: 3,
          label: '系统内部',
          children: [
            { id: 31, label: '运行日志 (L1)' },
            { id: 32, label: '关键配置 (L2)' }
          ]
        }
      ],
      tableData: [
        { id: 1, name: 'customer_name', category: '个人信息', sensitivity: 'L2', source: 'TCP:443', service: 'customer-gateway', permission: '合规', status: 'ok', statusLabel: '已合规', lastSeen: '2024-07-18 13:20', riskNote: '已通过脱敏策略校验', path: ['customer-gateway', 'data-bus', 'analytics-core'] },
        { id: 2, name: 'identity_number', category: '个人信息', sensitivity: 'L3', source: 'TCP:8443', service: 'identity-service', permission: '合规', status: 'todo', statusLabel: '待整改', lastSeen: '2024-07-18 13:18', riskNote: '接口调用频次高于基线，建议增加频控', path: ['identity-service', 'auth-center', 'risk-engine'] },
        { id: 3, name: 'bank_account', category: '个人信息', sensitivity: 'L4', source: 'TCP:9443', service: 'billing-service', permission: '超权限', status: 'violation', statusLabel: '违规', lastSeen: '2024-07-18 12:52', riskNote: '检测到非授权服务访问账单明细', path: ['billing-service', 'export-adapter', 'external-channel'] },
        { id: 4, name: 'dispatch_strategy', category: '企业机密', sensitivity: 'L4', source: 'TCP:7001', service: 'dispatch-scheduler', permission: '超权限', status: 'todo', statusLabel: '待整改', lastSeen: '2024-07-18 12:46', riskNote: '策略文件包含外部共享记录，需要确认权限', path: ['dispatch-scheduler', 'strategy-hub', 'partner-interface'] },
        { id: 5, name: 'grid_metrics_raw', category: '系统内部', sensitivity: 'L2', source: 'UDP:5002', service: 'grid-metrics', permission: '合规', status: 'ok', statusLabel: '已合规', lastSeen: '2024-07-18 13:05', riskNote: '数据按计划周期上传', path: ['grid-metrics', 'metrics-cache', 'analytics-core'] }
      ],
      selectedRecord: null,
      lineageChart: null
    }
  },
  computed: {
    filteredTableData() {
      return this.tableData.filter(item => {
        if (this.filters.keyword && !`${item.name}${item.category}${item.service}`.includes(this.filters.keyword)) return false
        if (this.filters.risk && this.filters.risk !== 'all' && item.sensitivity !== this.filters.risk) return false
        if (this.filters.compliance && this.filters.compliance !== 'all' && item.status !== this.filters.compliance) return false
        if (this.filters.onlyExceed && item.permission !== '超权限') return false
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
    statusTag(status) {
      switch (status) {
        case 'ok':
          return 'success'
        case 'todo':
          return 'warning'
        case 'violation':
          return 'danger'
        default:
          return 'info'
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
