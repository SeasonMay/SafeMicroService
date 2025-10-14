<template>
  <div class="app-container">
    <!-- 统计面板 -->
    <div class="stats-section">
      <div class="stat-card online">
        <div class="stat-icon">
          <i class="el-icon-success"></i>
        </div>
        <div class="stat-info">
          <div class="stat-number">{{ onlineCount }}</div>
          <div class="stat-label">在线设备</div>
        </div>
      </div>

      <div class="stat-card offline">
        <div class="stat-icon">
          <i class="el-icon-error"></i>
        </div>
        <div class="stat-info">
          <div class="stat-number">{{ offlineCount }}</div>
          <div class="stat-label">离线设备</div>
        </div>
      </div>

      <div class="stat-card total">
        <div class="stat-icon">
          <i class="el-icon-monitor"></i>
        </div>
        <div class="stat-info">
          <div class="stat-number">{{ totalDevices }}</div>
          <div class="stat-label">设备总数</div>
        </div>
      </div>

      <div class="stat-card temperature">
        <div class="stat-icon">
          <i class="el-icon-sunny"></i>
        </div>
        <div class="stat-info">
          <div class="stat-number">{{ averageTemperature }}°C</div>
          <div class="stat-label">平均温度</div>
        </div>
      </div>
    </div>

    <div class="refresh-section" style="margin-bottom: 24px;">
      <el-button type="primary" icon="el-icon-refresh" @click="refreshAll">刷新所有设备</el-button>
    </div>

    <!-- 设备卡片网格 -->
    <div class="devices-grid">
      <div
        v-for="device in deviceData"
        :key="device.id"
        class="device-card"
        :class="{ 'online': device.online, 'offline': !device.online }"
      >
        <!-- 卡片头部 -->
        <div class="card-header">
          <div class="device-avatar">
            <i class="el-icon-monitor"></i>
          </div>
          <div class="card-title">
            <h3>{{ device.name }}</h3>
            <span class="zone-tag">{{ device.zone }}</span>
          </div>
          <el-tag
            :type="device.online ? 'success' : 'danger'"
            class="status-tag"
            effect="dark"
            size="mini"
          >
            <i :class="device.online ? 'el-icon-success' : 'el-icon-error'"></i>
            {{ device.online ? '在线' : '离线' }}
          </el-tag>
        </div>

        <!-- 设备信息 -->
        <div class="card-content">
          <div class="info-grid">
            <div class="info-item">
              <i class="el-icon-location"></i>
              <div class="info-text">
                <span class="info-label">IP地址</span>
                <span class="info-value">{{ device.ip }}</span>
              </div>
            </div>

            <div class="info-item">
              <i class="el-icon-cpu"></i>
              <div class="info-text">
                <span class="info-label">设备型号</span>
                <span class="info-value">{{ device.model }}</span>
              </div>
            </div>
          </div>

          <!-- 在线设备显示实时系统信息 -->
          <div v-if="device.online" class="performance-section">
            <!-- CPU温度显示 -->
            <div v-if="device.temperature !== null" class="performance-item">
              <span class="perf-label">CPU温度</span>
              <div class="temperature-display">
                <span class="temp-value" :class="getTemperatureClass(device.temperature)">
                  {{ device.temperature }}°C
                </span>
                <i class="el-icon-sunny" :class="getTemperatureClass(device.temperature)"></i>
              </div>
            </div>

            <!-- 内存使用 -->
            <div class="performance-item">
              <span class="perf-label">内存</span>
              <el-progress
                :percentage="device.memory_percent || 0"
                :color="getLoadColor(device.memory_percent || 0)"
                :show-text="false"
                class="load-bar"
              ></el-progress>
              <span class="perf-value">{{ (device.memory_percent || 0).toFixed(1) }}%</span>
            </div>

            <!-- 硬盘存储 -->
            <div class="performance-item">
              <span class="perf-label">硬盘</span>
              <div class="storage-info">
                <div class="storage-usage">
                  <el-progress
                    :percentage="device.disk_percent || 0"
                    :color="getLoadColor(device.disk_percent || 0)"
                    :show-text="false"
                    class="load-bar"
                  ></el-progress>
                  <span class="perf-value">{{ (device.disk_percent || 0).toFixed(1) }}%</span>
                </div>
                <div class="storage-details">
                  <span class="storage-text">{{ formatStorageSpace(device.disk_used, device.disk_total) }}</span>
                </div>
              </div>
            </div>

            <div class="uptime-info">
              <i class="el-icon-timer"></i>
              <span>运行: {{ device.uptime || '未知' }}</span>
            </div>
          </div>

          <!-- 离线设备提示 -->
          <div v-else class="offline-info">
            <i class="el-icon-warning"></i>
            <span>设备离线</span>
          </div>
        </div>

        <!-- 操作按钮 -->
        <div class="card-actions">
          <el-button
            size="mini"
            type="primary"
            @click="pingDevice(device)"
            :loading="device.pinging"
            icon="el-icon-refresh"
          >
            {{ device.pinging ? '检测中' : 'Ping' }}
          </el-button>

          <el-button
            size="mini"
            type="info"
            @click="viewDetailedInfo(device)"
            :disabled="!device.online"
            icon="el-icon-info"
          >
            详情
          </el-button>
        </div>
      </div>
    </div>

    <!-- 详细信息对话框 -->
    <el-dialog
      title="设备详细信息"
      :visible.sync="detailDialogVisible"
      width="70%"
      :before-close="handleDetailClose"
    >
      <div v-if="selectedDevice" class="detail-content">
        <el-tabs v-model="activeTab" type="border-card">
          <!-- 系统概览 -->
          <el-tab-pane label="系统概览" name="overview">
            <div class="overview-grid">
              <div class="overview-item">
                <h4>基本信息</h4>
                <p><strong>设备名称:</strong> {{ selectedDevice.name }}</p>
                <p><strong>IP地址:</strong> {{ selectedDevice.ip }}</p>
                <p><strong>设备型号:</strong> {{ selectedDevice.model }}</p>
                <p><strong>操作系统:</strong> {{ selectedDevice.os }}</p>
                <p><strong>运行时间:</strong> {{ selectedDevice.uptime || '未知' }}</p>
                <p><strong>最后连接:</strong> {{ selectedDevice.online ? '现在' : formatTime(selectedDevice.lastSeen) }}</p>
              </div>

              <div class="overview-item">
                <h4>实时状态</h4>
                <p><strong>CPU温度:</strong>
                  <span :class="getTemperatureClass(selectedDevice.temperature)">
                    {{ selectedDevice.temperature ? selectedDevice.temperature + '°C' : 'N/A' }}
                  </span>
                </p>
                <p><strong>内存使用率:</strong> {{ (selectedDevice.memory_percent || 0).toFixed(1) }}%</p>
                <p><strong>磁盘使用率:</strong> {{ (selectedDevice.disk_percent || 0).toFixed(1) }}%</p>
                <p><strong>存储空间:</strong> {{ formatStorageSpace(selectedDevice.disk_used, selectedDevice.disk_total) }}</p>
                <p><strong>节点状态:</strong>
                  <el-tag :type="selectedDevice.online ? 'success' : 'danger'" size="mini">
                    {{ selectedDevice.online ? '在线' : '离线' }}
                  </el-tag>
                </p>
              </div>

              <div class="overview-item">
                <h4>性能图表</h4>
                <div class="chart-item">
                  <span>内存使用</span>
                  <el-progress
                    :percentage="selectedDevice.memory_percent || 0"
                    :color="getLoadColor(selectedDevice.memory_percent || 0)"
                  ></el-progress>
                </div>
                <div class="chart-item">
                  <span>磁盘使用</span>
                  <el-progress
                    :percentage="selectedDevice.disk_percent || 0"
                    :color="getLoadColor(selectedDevice.disk_percent || 0)"
                  ></el-progress>
                </div>
                <div class="chart-item">
                  <span>存储空间</span>
                  <div class="storage-chart">{{ formatStorageSpace(selectedDevice.disk_used, selectedDevice.disk_total) }}</div>
                </div>
              </div>

              <div class="overview-item">
                <h4>网络信息</h4>
                <p><strong>节点ID:</strong> {{ selectedDevice.node_id }}</p>
                <p><strong>HTTP端口:</strong> {{ getHttpPort(selectedDevice.node_id) }}</p>
                <p><strong>Socket端口:</strong> {{ getSocketPort(selectedDevice.ip) }}</p>
                <p><strong>区域:</strong> {{ selectedDevice.zone }}</p>
                <p><strong>连接状态:</strong> {{ selectedDevice.pinging ? '检测中...' : '就绪' }}</p>
              </div>
            </div>
          </el-tab-pane>

          <!-- 系统监控 -->
          <el-tab-pane label="系统监控" name="monitor">
            <div class="monitor-section">
              <div class="monitor-cards">
                <div class="monitor-card">
                  <div class="monitor-card-header">
                    <i class="el-icon-sunny"></i>
                    <h4>温度</h4>
                  </div>
                  <div class="monitor-card-content">
                    <div class="big-number" :class="getTemperatureClass(selectedDevice.temperature)">
                      {{ selectedDevice.temperature ? selectedDevice.temperature + '°C' : 'N/A' }}
                    </div>
                    <div class="temp-status">
                      {{ getTemperatureStatus(selectedDevice.temperature) }}
                    </div>
                  </div>
                </div>

                <div class="monitor-card">
                  <div class="monitor-card-header">
                    <i class="el-icon-s-data"></i>
                    <h4>内存</h4>
                  </div>
                  <div class="monitor-card-content">
                    <div class="big-number">{{ (selectedDevice.memory_percent || 0).toFixed(1) }}%</div>
                    <div class="monitor-progress">
                      <el-progress
                        :percentage="selectedDevice.memory_percent || 0"
                        :color="getLoadColor(selectedDevice.memory_percent || 0)"
                        :show-text="false"
                      ></el-progress>
                    </div>
                  </div>
                </div>

                <div class="monitor-card">
                  <div class="monitor-card-header">
                    <i class="el-icon-folder"></i>
                    <h4>硬盘存储</h4>
                  </div>
                  <div class="monitor-card-content">
                    <div class="big-number">{{ (selectedDevice.disk_percent || 0).toFixed(1) }}%</div>
                    <div class="storage-detail">{{ formatStorageSpace(selectedDevice.disk_used, selectedDevice.disk_total) }}</div>
                    <div class="monitor-progress">
                      <el-progress
                        :percentage="selectedDevice.disk_percent || 0"
                        :color="getLoadColor(selectedDevice.disk_percent || 0)"
                        :show-text="false"
                      ></el-progress>
                    </div>
                  </div>
                </div>
              </div>

              <div class="system-info">
                <h4>系统信息</h4>
                <div class="info-rows">
                  <div class="info-row">
                    <span class="info-key">系统运行时间:</span>
                    <span class="info-value">{{ selectedDevice.uptime || '未知' }}</span>
                  </div>
                  <div class="info-row">
                    <span class="info-key">最后更新时间:</span>
                    <span class="info-value">{{ formatUpdateTime(selectedDevice.last_update) }}</span>
                  </div>
                  <div class="info-row">
                    <span class="info-key">设备型号:</span>
                    <span class="info-value">{{ selectedDevice.model }}</span>
                  </div>
                  <div class="info-row">
                    <span class="info-key">操作系统:</span>
                    <span class="info-value">{{ selectedDevice.os }}</span>
                  </div>
                </div>
              </div>
            </div>
          </el-tab-pane>

          <!-- 连接测试 -->
          <el-tab-pane label="连接测试" name="test">
            <div class="test-section">
              <div class="test-card">
                <h4>网络连接测试</h4>
                <p>测试与设备的网络连接状况</p>
                <el-button
                  type="primary"
                  @click="pingDevice(selectedDevice)"
                  :loading="selectedDevice.pinging"
                  icon="el-icon-refresh"
                >
                  {{ selectedDevice.pinging ? '测试中...' : '开始Ping测试' }}
                </el-button>
              </div>

              <div class="test-card">
                <h4>系统信息刷新</h4>
                <p>手动刷新设备的系统监控信息</p>
                <el-button
                  type="success"
                  @click="refreshDetailedInfo"
                  icon="el-icon-refresh"
                >
                  刷新系统信息
                </el-button>
              </div>

              <div class="test-card">
                <h4>连接信息</h4>
                <div class="connection-info">
                  <p><strong>设备IP:</strong> {{ selectedDevice.ip }}</p>
                  <p><strong>HTTP端口:</strong> {{ getHttpPort(selectedDevice.node_id) }}</p>
                  <p><strong>Socket端口:</strong> {{ getSocketPort(selectedDevice.ip) }}</p>
                  <p><strong>节点类型:</strong> {{ getNodeType(selectedDevice.node_id) }}</p>
                </div>
              </div>
            </div>
          </el-tab-pane>
        </el-tabs>
      </div>

      <div slot="footer" class="dialog-footer">
        <el-button @click="detailDialogVisible = false">关闭</el-button>
        <el-button type="primary" @click="refreshDetailedInfo">刷新数据</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'DeviceMonitor',
  data() {
    return {
      detailDialogVisible: false,
      selectedDevice: null,
      activeTab: 'overview',
      systemInfoInterval: null,

      // 设备数据
      deviceData: [
        {
          id: 1,
          name: 'RaspberryPi-001',
          ip: '169.254.115.45',
          zone: '6203',
          online: false,
          lastSeen: new Date(),
          model: 'Pi 2B 1GB',
          os: 'Raspbian 10',
          uptime: null,
          pinging: false,
          node_id: 'client1',
          http_port: 8000,
          // 系统监控数据
          temperature: null,
          cpu_percent: 0,
          memory_percent: 0,
          disk_percent: 0,
          disk_used: 0,
          disk_total: 16.0, // 默认16GB总容量
          last_update: null
        },
        {
          id: 2,
          name: 'RaspberryPi-002',
          ip: '169.254.150.98',
          zone: '6203',
          online: false,
          lastSeen: new Date(),
          model: 'Pi 2B 1GB',
          os: 'Raspbian 10',
          uptime: null,
          pinging: false,
          node_id: 'mid1',
          http_port: 8002,
          temperature: null,
          cpu_percent: 0,
          memory_percent: 0,
          disk_percent: 0,
          disk_used: 0,
          disk_total: 16.0, // 默认16GB总容量
          last_update: null
        },
        {
          id: 3,
          name: 'RaspberryPi-003',
          ip: '169.254.135.218',
          zone: '6203',
          online: false,
          lastSeen: new Date(),
          model: 'Pi 2B 1GB',
          os: 'Raspbian 10',
          uptime: null,
          pinging: false,
          node_id: 'mid2',
          http_port: 8003,
          temperature: null,
          cpu_percent: 0,
          memory_percent: 0,
          disk_percent: 0,
          disk_used: 0,
          disk_total: 16.0, // 默认16GB总容量
          last_update: null
        },
        {
          id: 4,
          name: 'RaspberryPi-004',
          ip: '169.254.181.82',
          zone: '6203',
          online: false,
          lastSeen: new Date(),
          model: 'Pi 2B 1GB',
          os: 'Raspbian 10',
          uptime: null,
          pinging: false,
          node_id: 'mid3',
          http_port: 8004,
          temperature: null,
          cpu_percent: 0,
          memory_percent: 0,
          disk_percent: 0,
          disk_used: 0,
          disk_total: 16.0, // 默认16GB总容量
          last_update: null
        },
        {
          id: 5,
          name: 'RaspberryPi-005',
          ip: '169.254.139.96',
          zone: '6203',
          online: false,
          lastSeen: new Date(),
          model: 'Pi 2B 1GB',
          os: 'Raspbian 10',
          uptime: null,
          pinging: false,
          node_id: 'mid4',
          http_port: 8005,
          temperature: null,
          cpu_percent: 0,
          memory_percent: 0,
          disk_percent: 0,
          disk_used: 0,
          disk_total: 16.0, // 默认16GB总容量
          last_update: null
        },
        {
          id: 6,
          name: 'RaspberryPi-006',
          ip: '169.254.85.203',
          zone: '6203',
          online: false,
          lastSeen: new Date(),
          model: 'Pi 2B 1GB',
          os: 'Raspbian 10',
          uptime: null,
          pinging: false,
          node_id: 'client2',
          http_port: 8001,
          temperature: null,
          cpu_percent: 0,
          memory_percent: 0,
          disk_percent: 0,
          disk_used: 0,
          disk_total: 16.0, // 默认16GB总容量
          last_update: null
        }
      ]
    }
  },

  computed: {
    onlineCount() {
      return this.deviceData.filter(device => device.online).length
    },

    offlineCount() {
      return this.deviceData.filter(device => !device.online).length
    },

    totalDevices() {
      return this.deviceData.length
    },

    averageTemperature() {
      const onlineDevices = this.deviceData.filter(device => device.online && device.temperature !== null)
      if (onlineDevices.length === 0) return 'N/A'
      const sum = onlineDevices.reduce((acc, device) => acc + device.temperature, 0)
      return (sum / onlineDevices.length).toFixed(1)
    }
  },

  mounted() {
    // 启动时获取系统信息
    this.fetchAllSystemInfo()

    // 定期更新系统信息
    this.systemInfoInterval = setInterval(() => {
      this.fetchAllSystemInfo()
    }, 10000) // 每10秒更新一次
  },

  beforeDestroy() {
    if (this.systemInfoInterval) clearInterval(this.systemInfoInterval)
  },

  methods: {
    // 获取所有节点的系统信息
    async fetchAllSystemInfo() {
      console.log('开始获取所有节点系统信息...')

      // 并发获取所有设备的系统信息
      const promises = this.deviceData.map(device => this.fetchDeviceSystemInfo(device))

      try {
        await Promise.all(promises)
        console.log('所有节点系统信息获取完成')
      } catch (error) {
        console.error('获取系统信息时出现错误:', error)
      }
    },

    // 获取单个设备的系统信息
    async fetchDeviceSystemInfo(device) {
      try {
        const response = await fetch(`http://${device.ip}:${device.http_port}/api/system-info/basic`, {
          method: 'GET',
          timeout: 5000
        })

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`)
        }

        const result = await response.json()

        if (result.success && result.data) {
          // 更新设备状态为在线
          device.online = true
          device.last_update = new Date()
          device.lastSeen = new Date()

          // 更新系统信息
          const data = result.data
          device.temperature = data.temperature
          device.cpu_percent = data.cpu_percent || 0
          device.memory_percent = data.memory_percent || 0
          device.disk_percent = data.disk_percent || 0
          device.uptime = data.uptime || '未知'

          // 如果disk_percent有值但没有具体容量信息，根据百分比估算使用量
          if (device.disk_percent > 0 && device.disk_total > 0 && !device.disk_used) {
            device.disk_used = (device.disk_percent / 100) * device.disk_total
          }

          console.log(`✅ ${device.name} (${device.node_id}) 在线，温度: ${device.temperature}°C`)
        } else {
          this.setDeviceOffline(device, '数据格式错误')
        }
      } catch (error) {
        this.setDeviceOffline(device, error.message)
      }
    },

    // 设置设备为离线状态
    setDeviceOffline(device, reason) {
      device.online = false
      device.temperature = null
      device.cpu_percent = 0
      device.memory_percent = 0
      device.disk_percent = 0
      device.disk_used = 0
      // 保留总容量信息，不重置 disk_total
      device.uptime = null
      console.log(`❌ ${device.name} (${device.node_id}) 离线: ${reason}`)
    },

    // 获取设备详细系统信息
    async fetchDetailedSystemInfo(device) {
      try {
        const response = await fetch(`http://${device.ip}:${device.http_port}/api/system-info`, {
          method: 'GET',
          timeout: 10000
        })

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}`)
        }

        const result = await response.json()

        if (result.success && result.data) {
          const data = result.data

          // 更新基础信息
          device.temperature = data.temperature
          device.uptime = data.uptime

          // 更新CPU信息
          if (data.cpu) {
            device.cpu_percent = data.cpu.percent || 0
            device.cpu_count = data.cpu.count
            device.cpu_frequency = data.cpu.frequency
            device.load_1min = data.cpu.load_1min
            device.load_5min = data.cpu.load_5min
            device.load_15min = data.cpu.load_15min
          }

          // 更新内存信息
          if (data.memory) {
            device.memory_percent = data.memory.percent || 0
            device.memory_total = data.memory.total
            device.memory_used = data.memory.used
            device.memory_available = data.memory.available
          }

          // 更新磁盘信息
          if (data.disk) {
            device.disk_percent = data.disk.percent || 0
            device.disk_total = data.disk.total || device.disk_total // 保留默认值如果API没返回
            device.disk_used = data.disk.used || 0
            device.disk_free = data.disk.free
          }

          // 更新网络信息
          if (data.network) {
            device.network_bytes_sent = data.network.bytes_sent
            device.network_bytes_recv = data.network.bytes_recv
            device.network_packets_sent = data.network.packets_sent
            device.network_packets_recv = data.network.packets_recv
          }

          // 更新进程信息
          device.processes = data.processes || []

          console.log(`✅ ${device.name} 详细信息已更新`)
        }
      } catch (error) {
        console.error(`获取 ${device.name} 详细系统信息失败:`, error)
        this.$message.error(`获取 ${device.name} 详细信息失败: ${error.message}`)
      }
    },

    // 格式化时间
    formatTime(date) {
      if (!date) return '未知'
      return new Date(date).toLocaleString('zh-CN')
    },

    // 格式化更新时间
    formatUpdateTime(date) {
      if (!date) return '未知'
      const now = new Date()
      const diff = Math.floor((now - date) / 1000)

      if (diff < 60) return `${diff}秒前`
      if (diff < 3600) return `${Math.floor(diff / 60)}分钟前`
      if (diff < 86400) return `${Math.floor(diff / 3600)}小时前`
      return `${Math.floor(diff / 86400)}天前`
    },

    // 格式化存储空间
    formatStorageSpace(used, total) {
      if (!total) return 'N/A'
      const usedValue = used || 0
      return `${usedValue.toFixed(1)}GB / ${total.toFixed(1)}GB`
    },

    // 格式化字节数
    formatBytes(bytes) {
      if (!bytes) return 'N/A'
      const sizes = ['B', 'KB', 'MB', 'GB', 'TB']
      if (bytes === 0) return '0 B'
      const i = Math.floor(Math.log(bytes) / Math.log(1024))
      return Math.round(bytes / Math.pow(1024, i) * 100) / 100 + ' ' + sizes[i]
    },

    // 获取负载颜色
    getLoadColor(load) {
      if (load < 30) return '#67c23a'
      if (load < 70) return '#e6a23c'
      return '#f56c6c'
    },

    // 获取温度颜色类
    getTemperatureClass(temperature) {
      if (!temperature) return ''
      if (temperature < 50) return 'temp-normal'
      if (temperature < 70) return 'temp-warm'
      return 'temp-hot'
    },

    // 获取温度状态描述
    getTemperatureStatus(temperature) {
      if (!temperature) return '未知'
      if (temperature < 50) return '正常'
      if (temperature < 70) return '偏热'
      return '过热'
    },

    // 刷新所有设备
    refreshAll() {
      this.$message.success('正在刷新所有设备状态...')
      // 重置所有设备状态
      this.deviceData.forEach(device => {
        this.setDeviceOffline(device, '刷新中...')
      })
      // 重新获取信息
      this.fetchAllSystemInfo()
    },

    // Ping设备
    async pingDevice(device) {
      device.pinging = true
      const startTime = performance.now()

      try {
        const response = await fetch(`http://${device.ip}:${device.http_port}/test`, {
          method: 'GET',
          timeout: 5000
        })

        const endTime = performance.now()
        const responseTime = Math.round(endTime - startTime)

        device.pinging = false

        if (response.ok) {
          const result = await response.json()
          if (result.success) {
            this.$message.success(`${device.name} 连接成功，响应时间: ${responseTime}ms`)
            // 立即更新设备状态
            device.online = true
            device.lastSeen = new Date()
            // 获取最新系统信息
            await this.fetchDeviceSystemInfo(device)
          } else {
            this.$message.error(`${device.name} 连接失败: ${result.message || '未知错误'}`)
            this.setDeviceOffline(device, '连接失败')
          }
        } else {
          this.$message.error(`${device.name} 连接失败: HTTP ${response.status}`)
          this.setDeviceOffline(device, `HTTP ${response.status}`)
        }
      } catch (error) {
        device.pinging = false
        this.setDeviceOffline(device, error.message)
        this.$message.error(`${device.name} 连接失败: ${error.message}`)
      }
    },

    // 查看详细信息
    async viewDetailedInfo(device) {
      this.selectedDevice = device
      this.detailDialogVisible = true
      this.activeTab = 'overview'

      // 获取详细信息
      await this.fetchDetailedSystemInfo(device)
    },

    // 刷新详细信息
    async refreshDetailedInfo() {
      if (this.selectedDevice) {
        await this.fetchDetailedSystemInfo(this.selectedDevice)
        this.$message.success('详细信息已更新')
      }
    },

    // 关闭详细信息对话框
    handleDetailClose(done) {
      this.selectedDevice = null
      done()
    },

    // 获取HTTP端口
    getHttpPort(nodeId) {
      const device = this.deviceData.find(d => d.node_id === nodeId)
      return device ? device.http_port : '未知'
    },

    // 获取Socket端口
    getSocketPort(ip) {
      const portMap = {
        '169.254.115.45': 7000,
        '169.254.150.98': 7001,
        '169.254.135.218': 7002,
        '169.254.181.82': 7003,
        '169.254.139.96': 7004,
        '169.254.85.203': 7005
      }
      return portMap[ip] || '未知'
    },

    // 获取节点类型
    getNodeType(nodeId) {
      if (nodeId.includes('client')) return '客户端节点'
      if (nodeId.includes('mid')) return '中间转发节点'
      return '未知类型'
    }
  }
}
</script>

<style scoped>
.app-container {
  padding: 20px;
  background: #f5f7fa;
  min-height: 100vh;
}

/* 搜索区域 */
.refresh-section {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 统计面板 */
.stats-section {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
  margin-bottom: 24px;
}

.stat-card {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  align-items: center;
  transition: transform 0.2s ease;
}

.stat-card:hover {
  transform: translateY(-2px);
}

.stat-card.online .stat-icon {
  background: linear-gradient(135deg, #67c23a, #85ce61);
}

.stat-card.offline .stat-icon {
  background: linear-gradient(135deg, #f56c6c, #f78989);
}

.stat-card.total .stat-icon {
  background: linear-gradient(135deg, #409eff, #66b1ff);
}

.stat-card.temperature .stat-icon {
  background: linear-gradient(135deg, #e6a23c, #f7ba2a);
}

.stat-icon {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 20px;
  margin-right: 16px;
}

.stat-info {
  flex: 1;
}

.stat-number {
  font-size: 24px;
  font-weight: bold;
  color: #303133;
  line-height: 1;
}

.stat-label {
  font-size: 14px;
  color: #909399;
  margin-top: 4px;
}

/* 设备网格 */
.devices-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(380px, 1fr));
  gap: 20px;
}

.device-card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
  border: 2px solid transparent;
  overflow: hidden;
}

.device-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.15);
}

.device-card.online {
  border-left: 4px solid #67c23a;
}

.device-card.offline {
  border-left: 4px solid #f56c6c;
  opacity: 0.85;
}

/* 卡片头部 */
.card-header {
  display: flex;
  align-items: center;
  padding: 20px;
  background: linear-gradient(135deg, #fafbfc, #f1f3f4);
  border-bottom: 1px solid #f0f2f5;
}

.device-avatar {
  width: 48px;
  height: 48px;
  border-radius: 12px;
  background: linear-gradient(135deg, #409eff, #66b1ff);
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 20px;
  margin-right: 16px;
}

.card-title {
  flex: 1;
}

.card-title h3 {
  margin: 0 0 4px 0;
  font-size: 18px;
  color: #303133;
  font-weight: 600;
}

.zone-tag {
  font-size: 12px;
  color: #909399;
  background: #f5f7fa;
  padding: 2px 8px;
  border-radius: 4px;
}

.status-tag {
  margin-left: 12px;
}

/* 卡片内容 */
.card-content {
  padding: 20px;
}

.info-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
  margin-bottom: 16px;
}

.info-item {
  display: flex;
  align-items: center;
}

.info-item i {
  width: 20px;
  height: 20px;
  background: #f5f7fa;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #409eff;
  font-size: 12px;
  margin-right: 12px;
  flex-shrink: 0;
}

.info-text {
  flex: 1;
  min-width: 0;
}

.info-label {
  display: block;
  font-size: 12px;
  color: #909399;
  line-height: 1;
}

.info-value {
  display: block;
  font-size: 14px;
  color: #303133;
  font-weight: 500;
  margin-top: 2px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

/* 性能区域 */
.performance-section {
  background: #fafbfc;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 16px;
}

.performance-item {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 8px;
}

.performance-item:last-child {
  margin-bottom: 0;
}

.perf-label {
  font-size: 12px;
  color: #606266;
  min-width: 60px;
}

.load-bar {
  flex: 1;
}

.perf-value {
  font-size: 12px;
  color: #303133;
  font-weight: 500;
  min-width: 30px;
}

/* 硬盘存储信息 */
.storage-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.storage-usage {
  display: flex;
  align-items: center;
  gap: 12px;
}

.storage-details {
  margin-left: 0;
}

.storage-text {
  font-size: 10px;
  color: #909399;
  line-height: 1;
}

/* 温度显示 */
.temperature-display {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
  justify-content: flex-end;
}

.temp-value {
  font-size: 14px;
  font-weight: 600;
}

.temp-normal {
  color: #67c23a;
}

.temp-warm {
  color: #e6a23c;
}

.temp-hot {
  color: #f56c6c;
}

.uptime-info {
  display: flex;
  align-items: center;
  font-size: 12px;
  color: #606266;
  margin-top: 8px;
  padding-top: 8px;
  border-top: 1px solid #f0f2f5;
}

.uptime-info i {
  margin-right: 6px;
  color: #409eff;
}

/* 离线信息 */
.offline-info {
  background: #fef0f0;
  border: 1px solid #fbc4c4;
  border-radius: 6px;
  padding: 8px 12px;
  font-size: 12px;
  color: #f56c6c;
  margin-bottom: 16px;
}

.offline-info > div:first-child {
  display: flex;
  align-items: center;
}

.offline-info i {
  margin-right: 6px;
}

.offline-storage {
  margin-top: 4px;
  text-align: center;
}

.offline-storage .storage-text {
  font-size: 10px;
  color: #909399;
}

/* 操作按钮 */
.card-actions {
  display: flex;
  gap: 8px;
  padding: 0 20px 20px 20px;
}

.card-actions .el-button {
  flex: 1;
}

/* 详细信息对话框 */
.detail-content {
  min-height: 400px;
}

.overview-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 20px;
}

.overview-item {
  background: #fafbfc;
  border-radius: 8px;
  padding: 16px;
}

.overview-item h4 {
  margin: 0 0 12px 0;
  color: #303133;
  font-size: 16px;
  font-weight: 600;
  border-bottom: 2px solid #409eff;
  padding-bottom: 8px;
}

.overview-item p {
  margin: 8px 0;
  font-size: 14px;
  color: #606266;
}

.overview-item strong {
  color: #303133;
}

.chart-item {
  margin: 12px 0;
}

.chart-item span {
  display: block;
  font-size: 12px;
  color: #606266;
  margin-bottom: 4px;
}

/* 系统监控卡片 */
.monitor-section {
  padding: 20px 0;
}

.monitor-cards {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin-bottom: 30px;
}

.monitor-card {
  background: white;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  padding: 20px;
  text-align: center;
  transition: all 0.3s ease;
}

.monitor-card:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.monitor-card-header {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 15px;
}

.monitor-card-header i {
  font-size: 24px;
  margin-right: 8px;
  color: #409eff;
}

.monitor-card-header h4 {
  margin: 0;
  font-size: 16px;
  color: #303133;
}

.big-number {
  font-size: 32px;
  font-weight: bold;
  color: #303133;
  margin-bottom: 10px;
}

.monitor-progress {
  margin-top: 10px;
}

.temp-status {
  font-size: 12px;
  color: #909399;
}

.storage-detail {
  font-size: 11px;
  color: #606266;
  margin: 5px 0;
}

.storage-chart {
  font-size: 12px;
  color: #606266;
  padding: 4px 0;
}

.system-info {
  background: #fafbfc;
  border-radius: 8px;
  padding: 20px;
}

.system-info h4 {
  margin: 0 0 15px 0;
  color: #303133;
  font-size: 16px;
  font-weight: 600;
  border-bottom: 2px solid #409eff;
  padding-bottom: 8px;
}

.info-rows {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
}

.info-row {
  display: flex;
  justify-content: space-between;
  padding: 8px 0;
  border-bottom: 1px solid #f0f2f5;
}

.info-key {
  color: #606266;
  font-size: 14px;
}

.info-value {
  color: #303133;
  font-weight: 500;
  font-size: 14px;
}

/* 测试区域 */
.test-section {
  padding: 20px 0;
}

.test-card {
  background: #fafbfc;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  text-align: center;
}

.test-card h4 {
  margin: 0 0 10px 0;
  color: #303133;
  font-size: 16px;
}

.test-card p {
  margin: 0 0 15px 0;
  color: #606266;
  font-size: 14px;
}

.connection-info {
  text-align: left;
  background: white;
  border-radius: 6px;
  padding: 15px;
}

.connection-info p {
  margin: 8px 0;
  font-size: 14px;
  color: #606266;
}

.connection-info strong {
  color: #303133;
}

.no-data {
  text-align: center;
  padding: 40px;
  color: #909399;
}

.no-data i {
  font-size: 24px;
  margin-bottom: 8px;
  display: block;
}

/* 对话框样式 */
.dialog-footer {
  text-align: right;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .app-container {
    padding: 12px;
  }

  .devices-grid {
    grid-template-columns: 1fr;
  }

  .stats-section {
    grid-template-columns: repeat(2, 1fr);
  }

  .info-grid {
    grid-template-columns: 1fr;
  }

  .card-actions {
    flex-direction: column;
  }
}

@media (max-width: 480px) {
  .stats-section {
    grid-template-columns: 1fr;
  }

  .card-header {
    flex-wrap: wrap;
    gap: 12px;
  }

  .status-tag {
    margin-left: 0;
  }

  .performance-item {
    flex-direction: column;
    align-items: stretch;
    gap: 4px;
  }

  .perf-label {
    min-width: auto;
  }

  .temperature-display {
    justify-content: flex-start;
  }
}
</style>

