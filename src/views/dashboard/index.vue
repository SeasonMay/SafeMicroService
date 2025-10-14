<template>
  <div class="dashboard-container">
    <div class="dashboard-header">
      <div class="title-section">
        <h1 class="dashboard-title">密钥分发系统仪表盘</h1>
        <h3 class="dashboard-subtitle">用户: {{ name }}</h3>
      </div>
      <div class="control-panel">
        <el-input-number
          v-model="collectRounds"
          :min="10"
          :max="10000"
          :step="10"
          size="medium"
          placeholder="收集轮数"
          class="rounds-input"
        />
        <el-button
          type="primary"
          :loading="loading"
          class="collect-btn"
          @click="collectDelayData"
        >
          {{ loading ? '数据收集中...' : '获取时延分布' }}
        </el-button>
        <el-button
          v-if="loading"
          type="danger"
          class="stop-btn"
          @click="stopCollection"
        >
          停止收集
        </el-button>
        <el-button
          v-if="dataAvailable"
          type="info"
          class="load-btn"
          @click="loadFromFile"
        >
          从路径加载
        </el-button>
        <el-button
          v-if="dataAvailable"
          type="warning"
          class="clear-btn"
          @click="clearData"
        >
          清除数据
        </el-button>
        <!-- 实时监控控制 -->
        <el-divider direction="vertical" />
        <el-button
          :type="realTimeMonitoring ? 'danger' : 'success'"
          class="realtime-btn"
          :loading="realtimeLoading"
          @click="toggleRealTimeMonitoring"
        >
          {{ realTimeMonitoring ? '停止实时监控' : '开始实时监控' }}
        </el-button>
        <el-button
          v-if="realTimeMonitoring"
          type="info"
          class="config-btn"
          @click="showRealtimeConfig"
        >
          监控配置
        </el-button>
        <el-button
          type="default"
          class="refresh-btn"
          @click="forceRefreshChart"
        >
          刷新图表
        </el-button>
      </div>
    </div>

    <!-- 数据信息栏 -->
    <div v-if="dataAvailable" class="data-info">
      <el-alert
        :title="`数据来源: ${dataSource} | 获取时间: ${collectionTime}`"
        type="info"
        :closable="false"
      />
    </div>

    <!-- 实时监控状态 -->
    <div v-if="realTimeMonitoring" class="realtime-info">
      <el-alert
        :title="`实时监控中 | 最后更新: ${lastUpdateTime} | 监控时长: ${monitoringDuration}`"
        type="success"
        :closable="false"
      >
        <div class="realtime-stats-detail">
          <span>总测试: {{ realtimeStatistics.total_tests }}</span>
        </div>
      </el-alert>
    </div>

    <!-- 实时时延图表 - 默认显示 -->
    <div class="realtime-section">
      <el-card class="chart-card full-width">
        <div slot="header">
          <span>实时时延监控</span>
          <div class="realtime-stats">
            <span>Client1 最新: {{ latestClient1Delay }} ms</span>
            <span>Client2 最新: {{ latestClient2Delay }} ms</span>
          </div>
        </div>
        <div ref="realtimeChart" class="chart-container" />
      </el-card>
    </div>

    <div v-if="noDataAvailable && !realTimeMonitoring" class="no-data-message">
      <el-empty description="暂无时延数据，请点击上方按钮收集数据或从文件加载" />
    </div>

    <div v-if="dataAvailable" class="data-section">
      <!-- 统计概览 -->
      <div class="stats-overview">
        <el-card class="stats-card">
          <div slot="header">
            <span>Client1 数据概览 (1→2→1)</span>
          </div>
          <div class="stats-grid">
            <div class="stat-item">
              <div class="stat-value">{{ client1Stats.totalSamples }}</div>
              <div class="stat-label">收集样本数</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client1Stats.avgDelay }} ms</div>
              <div class="stat-label">平均时延</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client1Stats.minDelay }} ms</div>
              <div class="stat-label">最小时延</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client1Stats.maxDelay }} ms</div>
              <div class="stat-label">最大时延</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client1Stats.pathCount }}</div>
              <div class="stat-label">不同路径数</div>
            </div>
          </div>
        </el-card>

        <el-card class="stats-card">
          <div slot="header">
            <span>Client2 数据概览 (2→1→2)</span>
          </div>
          <div class="stats-grid">
            <div class="stat-item">
              <div class="stat-value">{{ client2Stats.totalSamples }}</div>
              <div class="stat-label">收集样本数</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client2Stats.avgDelay }} ms</div>
              <div class="stat-label">平均时延</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client2Stats.minDelay }} ms</div>
              <div class="stat-label">最小时延</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client2Stats.maxDelay }} ms</div>
              <div class="stat-label">最大时延</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ client2Stats.pathCount }}</div>
              <div class="stat-label">不同路径数</div>
            </div>
          </div>
        </el-card>
      </div>

      <!-- 时延序列图对比 -->
      <div class="charts-row">
        <el-card class="chart-card full-width">
          <div slot="header">
            <span>时延序列对比</span>
          </div>
          <div ref="timeSeriesChart" class="chart-container" />
        </el-card>
      </div>

      <!-- 时延分布图 -->
      <div class="charts-row">
        <el-card class="chart-card">
          <div slot="header">
            <span>Client1 时延分布</span>
          </div>
          <div ref="client1DistChart" class="chart-container" />
        </el-card>

        <el-card class="chart-card">
          <div slot="header">
            <span>Client2 时延分布</span>
          </div>
          <div ref="client2DistChart" class="chart-container" />
        </el-card>
      </div>

    </div>

    <!-- 实时监控配置对话框 -->
    <el-dialog
      title="实时监控配置"
      :visible.sync="configDialogVisible"
      width="500px"
    >
      <el-form :model="realtimeConfig" label-width="120px">
        <el-form-item label="测试间隔">
          <el-input-number
            v-model="realtimeConfig.test_interval"
            :min="2"
            :max="30"
            :step="1.0"
            suffix="秒"
          />
          <div class="config-help">每次测试的间隔时间</div>
        </el-form-item>

        <el-form-item label="超时时间">
          <el-input-number
            v-model="realtimeConfig.timeout"
            :min="10"
            :max="60"
            :step="5"
            suffix="秒"
          />
          <div class="config-help">单次测试的超时时间</div>
        </el-form-item>

        <el-form-item label="历史数据量">
          <el-input-number
            v-model="realtimeConfig.max_history"
            :min="100"
            :max="5000"
            :step="100"
          />
          <div class="config-help">保存的历史数据点数量</div>
        </el-form-item>

        <el-form-item label="重试次数">
          <el-input-number
            v-model="realtimeConfig.retry_count"
            :min="0"
            :max="3"
            :step="1"
          />
          <div class="config-help">测试失败时的重试次数</div>
        </el-form-item>
      </el-form>

      <div slot="footer" class="dialog-footer">
        <el-button @click="configDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="saveRealtimeConfig">保 存</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import * as echarts from 'echarts'

// 基础配置
const API_BASE_URL = 'http://localhost:5000/api'

// 树莓派节点配置 - 与后端保持一致
const RASPBERRY_NODES = {
  'client1': { 'ip': '169.254.115.45', 'port': 8000, 'name': 'RaspberryPi-001' },
  'client2': { 'ip': '169.254.85.203', 'port': 8001, 'name': 'RaspberryPi-006' },
  'mid1': { 'ip': '169.254.150.98', 'port': 8002, 'name': 'RaspberryPi-002' },
  'mid2': { 'ip': '169.254.135.218', 'port': 8003, 'name': 'RaspberryPi-003' },
  'mid3': { 'ip': '169.254.181.82', 'port': 8004, 'name': 'RaspberryPi-004' },
  'mid4': { 'ip': '169.254.139.96', 'port': 8005, 'name': 'RaspberryPi-005' }
}

export default {
  name: 'KeyDistributionBoard',
  data() {
    return {
      loading: false,
      collectRounds: 100,
      dataAvailable: false,
      noDataAvailable: false,
      client1Data: null,
      client2Data: null,
      collectionTime: '',
      dataSource: 'unknown',
      totalSamples: 0,
      currentFilePath: '',
      client1Stats: {
        totalSamples: 0,
        avgDelay: 0,
        minDelay: 0,
        maxDelay: 0,
        pathCount: 0
      },
      client2Stats: {
        totalSamples: 0,
        avgDelay: 0,
        minDelay: 0,
        maxDelay: 0,
        pathCount: 0
      },
      charts: {
        timeSeries: null,
        client1Dist: null,
        client2Dist: null,
        realtime: null
      },
      // 实时监控相关数据
      realTimeMonitoring: false,
      realtimeLoading: false,
      realtimeData: {
        client1: [],
        client2: [],
        timeLabels: []
      },
      realtimeTimer: null,
      lastUpdateTime: '',
      monitoringStartTime: null,
      monitoringDuration: '00:00:00',
      latestClient1Delay: '0.00',
      latestClient2Delay: '0.00',
      maxRealtimePoints: 100, // 最多显示100个数据点
      realtimeStatistics: {
        total_tests: 0,
        successful_tests: 0,
        failed_tests: 0
      },
      realtimeConfig: {
        test_interval: 1, // 增加到5秒间隔，给单次收集足够时间
        timeout: 2, // 单次收集超时时间
        max_history: 100,
        retry_count: 0
      },
      configDialogVisible: false
    }
  },
  computed: {
    ...mapGetters([
      'name'
    ])
  },
  mounted() {
    // 先检查数据可用性
    this.checkDataAvailability()

    // 延迟初始化实时图表，确保DOM完全渲染
    this.$nextTick(() => {
      setTimeout(() => {
        console.log('开始初始化实时图表...')
        this.initRealtimeChart()
        // 添加窗口大小变化监听
        window.addEventListener('resize', this.resizeAllCharts)
        console.log('组件挂载完成')
      }, 500)
    })
  },
  beforeDestroy() {
    // 清理定时器
    if (this.realtimeTimer) {
      clearInterval(this.realtimeTimer)
    }

    // 移除事件监听
    window.removeEventListener('resize', this.resizeAllCharts)

    // 销毁图表实例
    Object.values(this.charts).forEach(chart => {
      if (chart) {
        chart.dispose()
      }
    })
  },
  methods: {
    // ==================== 实时监控功能 ====================

    // 切换实时监控状态
    toggleRealTimeMonitoring() {
      if (this.realTimeMonitoring) {
        this.stopRealTimeMonitoring()
      } else {
        this.startRealTimeMonitoring()
      }
    },

    // 开始实时监控
    async startRealTimeMonitoring() {
      this.realtimeLoading = true

      try {
        this.realTimeMonitoring = true
        this.monitoringStartTime = new Date()
        this.realtimeData = {
          client1: [],
          client2: [],
          timeLabels: []
        }
        this.realtimeStatistics = {
          total_tests: 0,
          successful_tests: 0,
          failed_tests: 0
        }

        // 开始定时执行单次收集
        this.realtimeTimer = setInterval(() => {
          console.log('定时器触发，开始新一轮收集...')
          this.collectSingleRoundData()
          this.updateMonitoringDuration()
        }, this.realtimeConfig.test_interval * 1000) // 根据配置的间隔时间

        // 立即执行第一次收集
        setTimeout(() => {
          console.log('执行第一次收集...')
          this.collectSingleRoundData()
        }, 2000) // 延迟2秒执行第一次收集，确保图表已初始化

        this.$message({
          message: '实时监控已启动',
          type: 'success'
        })
      } catch (error) {
        console.error('启动实时监控失败:', error)
        this.$message({
          message: '启动实时监控失败',
          type: 'error'
        })
      } finally {
        this.realtimeLoading = false
      }
    },

    // 停止实时监控
    async stopRealTimeMonitoring() {
      this.realtimeLoading = true

      try {
        // 停止定时器
        if (this.realtimeTimer) {
          clearInterval(this.realtimeTimer)
          this.realtimeTimer = null
        }

        this.realTimeMonitoring = false

        this.$message({
          message: '实时监控已停止',
          type: 'info'
        })
      } catch (error) {
        console.error('停止实时监控失败:', error)
        this.realTimeMonitoring = false
      } finally {
        this.realtimeLoading = false
      }
    },

    // 收集单轮时延数据用于实时监控
    async collectSingleRoundData() {
      try {
        console.log('开始单轮时延收集...')

        // 1. 先停止所有现有收集任务
        await this.stopExistingCollections()

        // 等待一秒确保停止完成
        await new Promise(resolve => setTimeout(resolve, 1000))

        // 2. 启动单轮收集（1轮数据）
        const startPromises = [
          this.startClientCollection('client1', 1),
          this.startClientCollection('client2', 1)
        ]

        const startResults = await Promise.allSettled(startPromises)

        // 检查启动是否成功
        const allStarted = startResults.every(result =>
          result.status === 'fulfilled' && result.value.success
        )

        if (!allStarted) {
          console.log('部分客户端启动失败，跳过本次收集')
          this.realtimeStatistics.failed_tests++
          this.addFailedDataPoint()
          return
        }

        // 3. 等待收集完成并获取数据
        const maxWaitTime = this.realtimeConfig.timeout * 1000 // 转换为毫秒
        const startTime = Date.now()

        let client1Complete = false
        let client2Complete = false
        let client1Data = null
        let client2Data = null

        // 轮询检查完成状态
        while (Date.now() - startTime < maxWaitTime) {
          try {
            // 检查client1状态
            if (!client1Complete) {
              const client1Status = await this.getClientStatus('client1')
              if (!client1Status.is_collecting || client1Status.completed_rounds >= 1) {
                client1Complete = true
                client1Data = await this.getClientData('client1')
                console.log(`Client1完成，获取到${client1Data ? client1Data.length : 0}条数据`)
              }
            }

            // 检查client2状态
            if (!client2Complete) {
              const client2Status = await this.getClientStatus('client2')
              if (!client2Status.is_collecting || client2Status.completed_rounds >= 1) {
                client2Complete = true
                client2Data = await this.getClientData('client2')
                console.log(`Client2完成，获取到${client2Data ? client2Data.length : 0}条数据`)
              }
            }

            // 如果都完成了就退出循环
            if (client1Complete && client2Complete) {
              console.log('两个客户端都已完成收集')
              break
            }

            // 等待500ms后继续检查
            await new Promise(resolve => setTimeout(resolve, 500))
          } catch (error) {
            console.error('检查收集状态失败:', error)
            break
          }
        }

        // 4. 处理收集结果
        const now = new Date()
        const timeLabel = now.toLocaleTimeString()

        this.realtimeStatistics.total_tests++
        this.lastUpdateTime = timeLabel

        // 提取时延数据
        let client1Delay = null
        let client2Delay = null

        if (client1Data && client1Data.length > 0) {
          // 取最新的一条数据的时延
          const latestClient1 = client1Data[client1Data.length - 1]
          client1Delay = parseFloat(latestClient1.time) * 1000 // 转换为毫秒
          this.latestClient1Delay = client1Delay.toFixed(2)
          this.realtimeStatistics.successful_tests++
          console.log(`Client1时延: ${client1Delay.toFixed(2)}ms`)
        } else {
          console.log('Client1数据获取失败')
          this.realtimeStatistics.failed_tests++
        }

        if (client2Data && client2Data.length > 0) {
          // 取最新的一条数据的时延
          const latestClient2 = client2Data[client2Data.length - 1]
          client2Delay = parseFloat(latestClient2.time) * 1000 // 转换为毫秒
          this.latestClient2Delay = client2Delay.toFixed(2)
          console.log(`Client2时延: ${client2Delay.toFixed(2)}ms`)
        } else {
          console.log('Client2数据获取失败')
        }

        // 5. 更新图表数据
        this.realtimeData.client1.push(client1Delay)
        this.realtimeData.client2.push(client2Delay)
        this.realtimeData.timeLabels.push(timeLabel)

        // 限制数据点数量
        if (this.realtimeData.timeLabels.length > this.maxRealtimePoints) {
          this.realtimeData.client1.shift()
          this.realtimeData.client2.shift()
          this.realtimeData.timeLabels.shift()
        }

        // 更新图表
        this.updateRealtimeChart()

        console.log(`单轮收集完成: Client1=${client1Delay ? client1Delay.toFixed(2) : 'null'}ms, Client2=${client2Delay ? client2Delay.toFixed(2) : 'null'}ms`)
        console.log(`当前数据点数量: ${this.realtimeData.timeLabels.length}`)
      } catch (error) {
        console.error('单轮收集失败:', error)
        this.realtimeStatistics.failed_tests++
        this.addFailedDataPoint()
      }
    },

    // 添加失败数据点
    addFailedDataPoint() {
      const now = new Date()
      const timeLabel = now.toLocaleTimeString()

      console.log('添加失败数据点:', timeLabel)

      this.realtimeData.client1.push(null)
      this.realtimeData.client2.push(null)
      this.realtimeData.timeLabels.push(timeLabel)
      this.lastUpdateTime = timeLabel

      // 限制数据点数量
      if (this.realtimeData.timeLabels.length > this.maxRealtimePoints) {
        this.realtimeData.client1.shift()
        this.realtimeData.client2.shift()
        this.realtimeData.timeLabels.shift()
      }

      console.log('当前数据点数量(失败):', this.realtimeData.timeLabels.length)
      this.updateRealtimeChart()
    },

    // 停止现有收集任务
    async stopExistingCollections() {
      const stopPromises = ['client1', 'client2'].map(async(clientName) => {
        try {
          const node = RASPBERRY_NODES[clientName]
          const response = await fetch(`http://${node.ip}:${node.port}/stop-collection`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            signal: AbortSignal.timeout(3000)
          })
          console.log(`停止${clientName}: ${response.status}`)
          return { client: clientName, success: response.ok }
        } catch (error) {
          console.log(`停止${clientName}失败:`, error)
          return { client: clientName, success: false }
        }
      })

      const results = await Promise.allSettled(stopPromises)
      console.log('停止收集结果:', results)
    },

    // 启动客户端收集
    async startClientCollection(clientName, rounds) {
      try {
        const node = RASPBERRY_NODES[clientName]
        console.log(`启动${clientName}收集: ${node.ip}:${node.port}`)

        const response = await fetch(`http://${node.ip}:${node.port}/start-collection`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ rounds: rounds }),
          signal: AbortSignal.timeout(5000) // 5秒超时
        })

        if (response.status === 400) {
          // 如果客户端忙，尝试重置
          console.log(`${clientName}忙碌，尝试重置...`)
          try {
            const resetResponse = await fetch(`http://${node.ip}:${node.port}/reset`, {
              method: 'POST',
              signal: AbortSignal.timeout(3000)
            })

            if (resetResponse.ok) {
              // 重置成功，再次尝试启动
              await new Promise(resolve => setTimeout(resolve, 1000))
              const retryResponse = await fetch(`http://${node.ip}:${node.port}/start-collection`, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ rounds: rounds }),
                signal: AbortSignal.timeout(5000)
              })

              const retryResult = await retryResponse.json()
              console.log(`${clientName}重试结果:`, retryResult)
              return { success: retryResponse.ok, data: retryResult }
            }
          } catch (resetError) {
            console.error(`重置${clientName}失败:`, resetError)
          }
        }

        const result = await response.json()
        console.log(`${clientName}启动响应:`, result)
        return { success: response.ok, data: result }
      } catch (error) {
        console.error(`启动${clientName}收集失败:`, error)
        return { success: false, error: error.message }
      }
    },

    // 获取客户端状态
    async getClientStatus(clientName) {
      try {
        const node = RASPBERRY_NODES[clientName]
        const response = await fetch(`http://${node.ip}:${node.port}/collection-status`, {
          method: 'GET',
          signal: AbortSignal.timeout(3000)
        })

        if (response.ok) {
          const status = await response.json()
          console.log(`${clientName}状态:`, status)
          return status
        }

        return { is_collecting: false, completed_rounds: 0 }
      } catch (error) {
        console.error(`获取${clientName}状态失败:`, error)
        return { is_collecting: false, completed_rounds: 0 }
      }
    },

    // 获取客户端数据 - 复用原有方法
    async getClientData(clientName) {
      try {
        const node = RASPBERRY_NODES[clientName]
        const response = await fetch(`http://${node.ip}:${node.port}/get-data`, {
          method: 'GET',
          signal: AbortSignal.timeout(5000)
        })

        if (response.ok) {
          const responseData = await response.json()
          console.log(`${clientName}原始数据:`, responseData)

          if (clientName === 'client1') {
            // Client1返回格式: {"data": [{"path": "", "time": ""}]}
            const data = responseData.data || []
            console.log(`${clientName}处理后数据条数: ${data.length}`)
            return data
          } else {
            // Client2返回格式: {"success": true, "data": {"raw_data": [{"path": "", "delay": 0.123}]}}
            if (responseData.success) {
              const rawData = responseData.data?.raw_data || []
              // 转换为与client1相同的格式
              const convertedData = rawData.map(item => ({
                path: item.path || '',
                time: String(item.delay || 0)
              }))
              console.log(`${clientName}转换数据: ${rawData.length} -> ${convertedData.length}`)
              return convertedData
            }
            return []
          }
        }

        console.log(`${clientName}数据获取失败: ${response.status}`)
        return []
      } catch (error) {
        console.error(`获取${clientName}数据失败:`, error)
        return []
      }
    },

    // 显示实时监控配置对话框
    showRealtimeConfig() {
      this.configDialogVisible = true
    },

    // 强制刷新图表
    forceRefreshChart() {
      console.log('强制刷新图表，当前数据:', {
        timeLabels: this.realtimeData.timeLabels,
        client1: this.realtimeData.client1,
        client2: this.realtimeData.client2
      })

      if (this.charts.realtime) {
        this.charts.realtime.dispose()
        this.charts.realtime = null
      }

      this.$nextTick(() => {
        this.initRealtimeChart()
        if (this.realtimeData.timeLabels.length > 0) {
          this.updateRealtimeChart()
        }
      })
    },
    async saveRealtimeConfig() {
      try {
        this.configDialogVisible = false
        this.$message({
          message: '配置已保存',
          type: 'success'
        })

        // 如果正在监控，重新启动以应用新配置
        if (this.realTimeMonitoring) {
          this.$message({
            message: '配置已更新，监控将在下次循环时生效',
            type: 'info'
          })
        }
      } catch (error) {
        console.error('保存配置失败:', error)
        this.$message({
          message: '保存配置失败',
          type: 'error'
        })
      }
    },

    // 更新监控持续时间
    updateMonitoringDuration() {
      if (this.monitoringStartTime) {
        const now = new Date()
        const duration = now - this.monitoringStartTime
        const hours = Math.floor(duration / 3600000)
        const minutes = Math.floor((duration % 3600000) / 60000)
        const seconds = Math.floor((duration % 60000) / 1000)
        this.monitoringDuration = `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`
      }
    },

    // 初始化实时图表
    initRealtimeChart() {
      if (!this.$refs.realtimeChart) {
        console.log('实时图表DOM元素未就绪')
        return
      }

      if (this.charts.realtime) {
        this.charts.realtime.dispose()
      }

      try {
        this.charts.realtime = echarts.init(this.$refs.realtimeChart)

        const option = {
          title: {
            text: '实时时延监控',
            left: 'center',
            show: false
          },
          tooltip: {
            trigger: 'axis',
            formatter: function(params) {
              let result = `时间: ${params[0] ? params[0].axisValue : '无数据'}<br/>`
              params.forEach(param => {
                if (param.data !== null && param.data !== undefined) {
                  result += `${param.seriesName}: ${param.data.toFixed(2)} ms<br/>`
                } else {
                  result += `${param.seriesName}: 无数据<br/>`
                }
              })
              return result
            }
          },
          legend: {
            data: ['Client1 实时时延', 'Client2 实时时延'],
            top: 10
          },
          grid: {
            left: '3%',
            right: '4%',
            bottom: '10%',
            top: '15%',
            containLabel: true
          },
          xAxis: {
            type: 'category',
            data: [], // 初始为空数组
            axisLabel: {
              rotate: 45,
              fontSize: 10
            },
            boundaryGap: false
          },
          yAxis: {
            type: 'value',
            name: '时延 (ms)',
            scale: true,
            min: 0,
            axisLabel: {
              formatter: '{value} ms'
            }
          },
          dataZoom: [
            {
              type: 'inside',
              start: 0,
              end: 100
            }
          ],
          series: [
            {
              name: 'Client1 实时时延',
              data: [], // 初始为空数组
              type: 'line',
              symbol: 'circle',
              symbolSize: 6,
              lineStyle: {
                width: 2
              },
              itemStyle: {
                color: '#5470c6'
              },
              connectNulls: false,
              smooth: false // 改为false避免数据少时的问题
            },
            {
              name: 'Client2 实时时延',
              data: [], // 初始为空数组
              type: 'line',
              symbol: 'circle',
              symbolSize: 6,
              lineStyle: {
                width: 2
              },
              itemStyle: {
                color: '#ee6666'
              },
              connectNulls: false,
              smooth: false // 改为false避免数据少时的问题
            }
          ]
        }

        this.charts.realtime.setOption(option, true) // 使用notMerge=true确保完全重新设置

        // 确保图表正确渲染
        setTimeout(() => {
          if (this.charts.realtime) {
            this.charts.realtime.resize()
          }
        }, 100)

        console.log('实时图表初始化成功')
      } catch (error) {
        console.error('初始化实时图表失败:', error)
      }
    },

    // 更新实时图表
    updateRealtimeChart() {
      if (!this.charts.realtime) {
        console.log('实时图表未初始化，重新初始化...')
        this.initRealtimeChart()
        return
      }

      try {
        // 确保数据存在
        if (!this.realtimeData.timeLabels || this.realtimeData.timeLabels.length === 0) {
          console.log('没有时间标签数据，跳过更新')
          return
        }

        console.log('更新图表数据:', {
          timeLabels: this.realtimeData.timeLabels.length,
          client1: this.realtimeData.client1.length,
          client2: this.realtimeData.client2.length,
          latestClient1: this.realtimeData.client1[this.realtimeData.client1.length - 1],
          latestClient2: this.realtimeData.client2[this.realtimeData.client2.length - 1]
        })

        const option = {
          xAxis: {
            data: [...this.realtimeData.timeLabels] // 创建新数组避免引用问题
          },
          series: [
            {
              data: [...this.realtimeData.client1] // 创建新数组避免引用问题
            },
            {
              data: [...this.realtimeData.client2] // 创建新数组避免引用问题
            }
          ]
        }

        this.charts.realtime.setOption(option, false) // notMerge=false，只更新数据

        console.log('图表更新成功')
      } catch (error) {
        console.error('更新实时图表失败:', error)
        // 如果更新失败，尝试重新初始化
        setTimeout(() => {
          this.initRealtimeChart()
        }, 1000)
      }
    },

    // ==================== 原有方法保持不变 ====================

    // 生成CSV内容
    generateCSVContent(data, client) {
      const headers = ['序号', '传输路径', '时延(ms)']
      const csvContent = [
        headers.join(','),
        ...data.map((item, index) => [
          index + 1,
          item.path,
          (parseFloat(item.time) * 1000).toFixed(3)
        ].join(','))
      ].join('\n')

      return csvContent
    },

    // 自动保存到本地路径（临时方案：使用浏览器下载）
    async autoSaveToLocal() {
      try {
        console.log('开始自动保存数据...')

        // 生成CSV文件内容（固定文件名）
        const client1CSV = this.generateCSVContent(this.client1Data, 'client1')
        const client2CSV = this.generateCSVContent(this.client2Data, 'client2')

        // 方案1：尝试通过后端API保存
        try {
          const saveResult = await this.saveToServer('E:\\DelayKeyGeneration\\delays', client1CSV, client2CSV)

          if (saveResult.success) {
            this.currentFilePath = `E:\\DelayKeyGeneration\\delays`
            console.log('通过后端API保存成功')
            this.$message({
              message: `数据已自动保存到: E:\\DelayKeyGeneration\\delays`,
              type: 'success',
              duration: 3000
            })
            return
          }
        } catch (apiError) {
          console.log('后端API保存失败，尝试浏览器下载方案', apiError)
        }

        // 方案2：使用浏览器下载（备用方案）
        this.downloadFile('client1_delay.csv', client1CSV, 'text/csv')
        this.downloadFile('client2_delay.csv', client2CSV, 'text/csv')

        this.currentFilePath = '浏览器下载目录'
        console.log('使用浏览器下载方案保存')
        this.$message({
          message: '数据已下载到浏览器下载目录（请手动移动到目标文件夹）',
          type: 'warning',
          duration: 5000
        })
      } catch (error) {
        console.error('自动保存失败:', error)
        this.$message({
          message: `自动保存失败: ${error.message}`,
          type: 'error'
        })
      }
    },

    // 通过后端API保存文件到服务器本地
    async saveToServer(basePath, client1CSV, client2CSV) {
      try {
        const response = await fetch(`${API_BASE_URL}/save-files`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            basePath: basePath,
            client1Data: client1CSV,
            client2Data: client2CSV,
            client1Filename: 'client1_delay.csv',
            client2Filename: 'client2_delay.csv'
          })
        })

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`)
        }

        const result = await response.json()
        return result
      } catch (error) {
        console.error('调用保存API失败:', error)
        throw error
      }
    },

    // 下载文件到浏览器
    downloadFile(filename, content, type) {
      const blob = new Blob([content], { type })
      const link = document.createElement('a')
      link.href = URL.createObjectURL(blob)
      link.download = filename
      link.click()
      URL.revokeObjectURL(link.href)
    },

    // 从指定路径的CSV文件加载数据
    async loadFromFile() {
      try {
        const response = await fetch(`${API_BASE_URL}/load-from-path`, {
          method: 'GET',
          headers: {
            'Content-Type': 'application/json'
          }
        })

        const result = await response.json()

        if (result.success && result.data) {
          this.client1Data = result.data.client1Data
          this.client2Data = result.data.client2Data
          this.collectionTime = result.data.loadTime || '从路径文件加载'
          this.dataSource = '路径文件加载'
          this.currentFilePath = result.data.basePath

          this.dataAvailable = true
          this.noDataAvailable = false
          this.calculateStats()
          this.$nextTick(() => {
            this.initCharts()
          })

          this.$message({
            message: `数据已加载，共${this.totalSamples}个样本`,
            type: 'success'
          })
        } else {
          this.$message({
            message: result.message || '指定路径下没有找到CSV文件',
            type: 'warning'
          })
        }
      } catch (error) {
        console.error('从路径加载文件失败:', error)
        this.$message({
          message: '加载文件失败，请检查网络连接',
          type: 'error'
        })
      }
    },

    // 从对象加载数据
    loadDataFromObject(data) {
      if (data.client1Data && data.client2Data) {
        this.client1Data = data.client1Data
        this.client2Data = data.client2Data
        this.collectionTime = data.collectionInfo?.collectionTime || '从文件加载'
        this.dataSource = '文件加载'
        this.currentFilePath = data.collectionInfo?.storagePath || ''

        this.dataAvailable = true
        this.noDataAvailable = false
        this.calculateStats()
        this.$nextTick(() => {
          this.initCharts()
        })
      }
    },

    // 检查数据可用性 - 优先检查指定路径的CSV文件
    async checkDataAvailability() {
      // 1. 首先检查指定路径的CSV文件
      try {
        const response = await fetch(`${API_BASE_URL}/load-from-path`, {
          method: 'GET',
          headers: {
            'Content-Type': 'application/json'
          }
        })

        if (response.ok) {
          const result = await response.json()
          if (result.success && result.data) {
            this.client1Data = result.data.client1Data
            this.client2Data = result.data.client2Data
            this.collectionTime = result.data.loadTime
            this.dataSource = '路径CSV文件'
            this.currentFilePath = result.data.basePath

            this.dataAvailable = true
            this.noDataAvailable = false
            this.calculateStats()
            this.$nextTick(() => {
              this.initCharts()
            })
            console.log('使用指定路径的CSV文件数据')
            return
          }
        }
      } catch (error) {
        console.log('指定路径无CSV文件，检查后端默认位置', error)
      }

      // 2. 检查后端默认位置的CSV文件
      try {
        const response = await fetch(`${API_BASE_URL}/check-data`, {
          method: 'GET',
          headers: {
            'Content-Type': 'application/json'
          }
        })

        const result = await response.json()

        if (result.hasData) {
          this.client1Data = result.client1Data
          this.client2Data = result.client2Data
          this.collectionTime = '从后端默认CSV文件加载'
          this.dataSource = '后端默认CSV文件'
          this.currentFilePath = '后端默认位置'

          this.dataAvailable = true
          this.noDataAvailable = false
          this.calculateStats()
          this.$nextTick(() => {
            this.initCharts()
          })
          console.log('从后端默认CSV文件加载数据')
        } else {
          this.dataAvailable = false
          this.noDataAvailable = true
          console.log('无可用数据')
        }
      } catch (error) {
        console.error('检查数据失败:', error)
        this.noDataAvailable = true
      }
    },

    // 收集时延数据
    async collectDelayData() {
      this.loading = true

      try {
        const startResponse = await fetch(`${API_BASE_URL}/start-collection`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            rounds: this.collectRounds
          })
        })

        const startResult = await startResponse.json()

        if (!startResult.success) {
          this.$message({
            message: startResult.message || '启动收集失败',
            type: 'error'
          })
          this.loading = false
          return
        }

        this.$message({
          message: `正在收集时延数据，轮数：${this.collectRounds}`,
          type: 'info'
        })

        this.pollCollectionStatus()
      } catch (error) {
        console.error('收集数据失败:', error)
        this.$message({
          message: '网络错误，无法连接到服务器',
          type: 'error'
        })
        this.loading = false
      }
    },

    // 轮询收集状态
    async pollCollectionStatus() {
      const pollInterval = setInterval(async() => {
        try {
          const response = await fetch(`${API_BASE_URL}/collection-status`, {
            method: 'GET'
          })

          const result = await response.json()

          if (result.is_collecting && result.completed_rounds > 0) {
            this.$message({
              message: `收集进度: ${result.completed_rounds}/${result.total_rounds}`,
              type: 'info',
              duration: 1000
            })
          }

          if (result.complete) {
            clearInterval(pollInterval)
            this.loading = false

            if (result.results && !result.results.error) {
              this.$message({
                message: '时延数据收集成功！',
                type: 'success'
              })

              this.client1Data = result.results.client1Data
              this.client2Data = result.results.client2Data
              this.collectionTime = new Date().toLocaleString('zh-CN')
              this.dataSource = 'API收集'

              this.dataAvailable = true
              this.noDataAvailable = false

              this.calculateStats()
              this.$nextTick(() => {
                this.initCharts()
              })
            } else {
              this.$message({
                message: result.results?.error || '收集过程中出现错误',
                type: 'error'
              })
            }
          }
        } catch (error) {
          console.error('轮询状态失败:', error)
        }
      }, 1000)

      const maxPollTime = (this.collectRounds * 1.5 + 60) * 1000
      setTimeout(() => {
        clearInterval(pollInterval)
        if (this.loading) {
          this.loading = false
          this.$message({
            message: '收集超时，请检查网络连接',
            type: 'error'
          })
        }
      }, maxPollTime)
    },

    // 停止收集
    async stopCollection() {
      try {
        const response = await fetch(`${API_BASE_URL}/stop-collection`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          }
        })

        const result = await response.json()

        if (result.success) {
          this.$message({
            message: '收集已停止',
            type: 'success'
          })
          this.loading = false
        } else {
          this.$message({
            message: result.message || '停止失败',
            type: 'error'
          })
        }
      } catch (error) {
        console.error('停止收集失败:', error)
        this.$message({
          message: '网络错误',
          type: 'error'
        })
      }
    },

    // 清除数据
    clearData() {
      this.$confirm('确定要清除所有时延数据吗？', '确认清除', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        this.client1Data = null
        this.client2Data = null
        this.dataAvailable = false
        this.noDataAvailable = true
        this.collectionTime = ''
        this.dataSource = 'unknown'
        this.totalSamples = 0
        this.currentFilePath = ''

        Object.values(this.charts).forEach(chart => {
          if (chart && chart !== this.charts.realtime) {
            chart.dispose()
          }
        })
        this.charts.timeSeries = null
        this.charts.client1Dist = null
        this.charts.client2Dist = null

        this.$message({
          message: '数据已清除',
          type: 'success'
        })
      }).catch(() => {
        // 用户取消
      })
    },

    // 计算统计数据
    calculateStats() {
      if (this.client1Data) {
        const times = this.client1Data.map(item => parseFloat(item.time))
        const paths = new Set(this.client1Data.map(item => item.path))

        this.client1Stats = {
          totalSamples: this.client1Data.length,
          avgDelay: (times.reduce((a, b) => a + b, 0) / times.length * 1000).toFixed(2),
          minDelay: (Math.min(...times) * 1000).toFixed(2),
          maxDelay: (Math.max(...times) * 1000).toFixed(2),
          pathCount: paths.size
        }
      }

      if (this.client2Data) {
        const times = this.client2Data.map(item => parseFloat(item.time))
        const paths = new Set(this.client2Data.map(item => item.path))

        this.client2Stats = {
          totalSamples: this.client2Data.length,
          avgDelay: (times.reduce((a, b) => a + b, 0) / times.length * 1000).toFixed(2),
          minDelay: (Math.min(...times) * 1000).toFixed(2),
          maxDelay: (Math.max(...times) * 1000).toFixed(2),
          pathCount: paths.size
        }
      }

      this.totalSamples = (this.client1Data?.length || 0) + (this.client2Data?.length || 0)
    },

    // 初始化所有图表
    initCharts() {
      this.initTimeSeriesChart()
      this.initDistributionCharts()
      window.addEventListener('resize', this.resizeAllCharts)
    },

    // 初始化时延序列对比图
    initTimeSeriesChart() {
      if (this.charts.timeSeries) {
        this.charts.timeSeries.dispose()
      }
      this.charts.timeSeries = echarts.init(this.$refs.timeSeriesChart)

      const client1TimeData = this.client1Data.map((item, index) => [index, parseFloat(item.time) * 1000])
      const client2TimeData = this.client2Data.map((item, index) => [index, parseFloat(item.time) * 1000])

      const option = {
        tooltip: {
          trigger: 'axis',
          formatter: function(params) {
            let result = `样本序号: ${params[0].dataIndex}<br/>`
            params.forEach(param => {
              result += `${param.seriesName}: ${param.data[1].toFixed(2)} ms<br/>`
            })
            return result
          }
        },
        legend: {
          data: ['Client1 (1→2→1)', 'Client2 (2→1→2)']
        },
        xAxis: {
          type: 'value',
          name: '样本序号'
        },
        yAxis: {
          type: 'value',
          name: '时延 (ms)'
        },
        series: [
          {
            name: 'Client1 (1→2→1)',
            data: client1TimeData,
            type: 'line',
            symbolSize: 4,
            symbol: 'circle',
            sampling: 'lttb',
            itemStyle: {
              color: '#5470c6'
            },
            lineStyle: {
              width: 2
            }
          },
          {
            name: 'Client2 (2→1→2)',
            data: client2TimeData,
            type: 'line',
            symbolSize: 4,
            symbol: 'circle',
            sampling: 'lttb',
            itemStyle: {
              color: '#ee6666'
            },
            lineStyle: {
              width: 2
            }
          }
        ]
      }

      this.charts.timeSeries.setOption(option)
    },

    // 初始化时延分布图
    initDistributionCharts() {
      // Client1 分布图
      if (this.charts.client1Dist) {
        this.charts.client1Dist.dispose()
      }
      this.charts.client1Dist = echarts.init(this.$refs.client1DistChart)

      const client1Delays = this.client1Data.map(item => parseFloat(item.time) * 1000)
      const client1DistData = this.calculateDistribution(client1Delays)

      this.charts.client1Dist.setOption({
        tooltip: {
          trigger: 'item',
          formatter: function(params) {
            return `时延范围: ${params.name} ms<br/>样本数: ${params.value}`
          }
        },
        xAxis: {
          type: 'category',
          data: client1DistData.labels,
          axisLabel: {
            rotate: 45,
            fontSize: 10
          }
        },
        yAxis: {
          type: 'value',
          name: '样本数量'
        },
        series: [{
          data: client1DistData.values,
          type: 'bar',
          itemStyle: {
            color: '#5470c6'
          }
        }]
      })

      // Client2 分布图
      if (this.charts.client2Dist) {
        this.charts.client2Dist.dispose()
      }
      this.charts.client2Dist = echarts.init(this.$refs.client2DistChart)

      const client2Delays = this.client2Data.map(item => parseFloat(item.time) * 1000)
      const client2DistData = this.calculateDistribution(client2Delays)

      this.charts.client2Dist.setOption({
        tooltip: {
          trigger: 'item',
          formatter: function(params) {
            return `时延范围: ${params.name} ms<br/>样本数: ${params.value}`
          }
        },
        xAxis: {
          type: 'category',
          data: client2DistData.labels,
          axisLabel: {
            rotate: 45,
            fontSize: 10
          }
        },
        yAxis: {
          type: 'value',
          name: '样本数量'
        },
        series: [{
          data: client2DistData.values,
          type: 'bar',
          itemStyle: {
            color: '#ee6666'
          }
        }]
      })
    },

    // 计算分布数据
    calculateDistribution(delays) {
      const min = Math.min(...delays)
      const max = Math.max(...delays)
      const range = max - min
      const binCount = 20
      const binSize = range / binCount

      const bins = Array(binCount).fill(0)
      delays.forEach(delay => {
        const binIndex = Math.min(Math.floor((delay - min) / binSize), binCount - 1)
        bins[binIndex]++
      })

      const labels = Array(binCount).fill(0).map((_, i) => {
        return `${(min + i * binSize).toFixed(1)}-${(min + (i + 1) * binSize).toFixed(1)}`
      })

      return { labels, values: bins }
    },

    // 调整所有图表大小
    resizeAllCharts() {
      for (const chart of Object.values(this.charts)) {
        if (chart) {
          try {
            chart.resize()
          } catch (error) {
            console.error('调整图表大小失败:', error)
          }
        }
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.dashboard {
  &-container {
    padding: 30px;
    background-color: #f5f7fa;
    min-height: calc(100vh - 100px);
  }

  &-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 30px;
  }

  &-title {
    font-size: 28px;
    color: #303133;
    margin: 0;
    padding: 0;
  }

  &-subtitle {
    font-size: 16px;
    color: #606266;
    margin: 8px 0 0;
    font-weight: normal;
  }
}

.title-section {
  flex: 1;
}

.control-panel {
  display: flex;
  gap: 15px;
  align-items: center;
  flex-wrap: wrap;
}

.path-input {
  width: 200px;
}

.rounds-input {
  width: 140px;
}

.collect-btn {
  width: 140px;
}

.stop-btn {
  width: 100px;
}

.load-btn {
  width: 100px;
}

.clear-btn {
  width: 100px;
}

.realtime-btn {
  width: 140px;
}

.config-btn {
  width: 100px;
}

.realtime-stats-detail {
  margin-top: 8px;
  font-size: 12px;

  span {
    margin-right: 20px;
    color: #606266;
  }
}

.config-help {
  font-size: 12px;
  color: #909399;
  margin-top: 5px;
}

.data-info {
  margin-bottom: 20px;
}

.realtime-info {
  margin-bottom: 20px;
}

.realtime-section {
  margin-bottom: 30px;
}

.realtime-stats {
  float: right;
  font-size: 14px;
  color: #606266;

  span {
    margin-left: 20px;
  }
}

.no-data-message {
  margin-top: 60px;
  text-align: center;
}

.data-section {
  margin-top: 30px;
}

/* 统计概览样式 */
.stats-overview {
  display: flex;
  gap: 20px;
  margin-bottom: 30px;
}

.stats-card {
  flex: 1;
  background-color: #fff;
  border-radius: 4px;
}

.stats-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}

.stat-item {
  flex: 1;
  min-width: 120px;
  text-align: center;
  padding: 10px;
  background-color: #f8f9fa;
  border-radius: 4px;
}

.stat-value {
  font-size: 20px;
  color: #303133;
  font-weight: bold;
  margin-bottom: 5px;
}

.stat-label {
  font-size: 12px;
  color: #909399;
  margin: 0;
}

/* 图表布局 */
.charts-row {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
}

.chart-card {
  flex: 1;
  background-color: #fff;
  border-radius: 4px;

  &.full-width {
    width: 100%;
  }
}

.chart-container {
  height: 400px;
  width: 100%;
  min-height: 400px;
}

@media screen and (max-width: 768px) {
  .dashboard-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 15px;
  }

  .control-panel {
    width: 100%;
    justify-content: flex-start;
  }

  .path-input {
    width: 100%;
    margin-bottom: 10px;
  }

  .stats-overview {
    flex-direction: column;
  }

  .charts-row {
    flex-direction: column;
  }

  .chart-container {
    height: 300px;
  }

  .stats-grid {
    flex-direction: column;
    gap: 10px;
  }

  .stat-item {
    min-width: unset;
  }
}
</style>
