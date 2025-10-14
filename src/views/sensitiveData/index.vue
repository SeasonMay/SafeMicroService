<template>
  <div class="app-container">
    <div class="filter-container">
      <el-button class="filter-item" type="primary" icon="el-icon-refresh" @click="fetchData">
        刷新数据
      </el-button>
      <el-button class="filter-item" type="success" icon="el-icon-folder-opened" @click="openKeyDirectory">
        密钥目录
      </el-button>
    </div>

    <el-table
      v-loading="listLoading"
      :data="list"
      element-loading-text="加载中..."
      border
      fit
      highlight-current-row
    >
      <el-table-column align="center" label="ID" width="80">
        <template v-slot="scope">
          {{ scope.$index + 1 }}
        </template>
      </el-table-column>
      <el-table-column label="密钥文件名" width="280">
        <template v-slot="scope">
          <span class="link-type" @click="viewDetails(scope.row)">{{ scope.row.filename }}</span>
        </template>
      </el-table-column>
      <el-table-column label="适用加密算法" width="120" align="center">
        <template v-slot="scope">
          <el-tag :type="scope.row.algorithm | keyTypeFilter">
            {{ scope.row.algorithm }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="密钥长度" width="100" align="center">
        <template v-slot="scope">
          {{ scope.row.key_length }}
        </template>
      </el-table-column>
      <el-table-column label="量化位数" width="100" align="center">
        <template v-slot="scope">
          <el-tag type="info" size="small">{{ scope.row.quantization }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column label="生成时间" width="180" align="center">
        <template v-slot="scope">
          <i class="el-icon-time" />
          <span>{{ scope.row.generation_time }}</span>
        </template>
      </el-table-column>
      <el-table-column label="结束时间" width="180" align="center">
        <template v-slot="scope">
          <i class="el-icon-clock" />
          <span>{{ scope.row.end_time }}</span>
        </template>
      </el-table-column>
      <el-table-column label="状态" width="100" align="center">
        <template v-slot="scope">
          <el-tag :type="getStatusType(scope.row.end_time)">
            {{ getStatusText(scope.row.end_time) }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="备注说明" width="150">
        <template v-slot="scope">
          <span>{{ scope.row.description || '无' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" width="260" align="center">
        <template v-slot="scope">
          <el-button
            size="mini"
            type="success"
            @click="copyKey(scope.row)"
          >
            复制密钥
          </el-button>
          <el-button
            size="mini"
            type="danger"
            @click="deleteKey(scope.row)"
          >
            删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 详情对话框 -->
    <el-dialog
      title="密钥详情"
      :visible.sync="detailDialogVisible"
      width="60%"
    >
      <div v-if="selectedKey">
        <el-descriptions title="基本信息" :column="2" border>
          <el-descriptions-item label="文件名">{{ selectedKey.filename }}</el-descriptions-item>
          <el-descriptions-item label="加密算法">{{ selectedKey.keyInfo.加密算法 }}</el-descriptions-item>
          <el-descriptions-item label="密钥长度">{{ selectedKey.keyInfo.密钥长度 }}</el-descriptions-item>
          <el-descriptions-item label="量化位数">{{ selectedKey.keyInfo.量化位数 }}</el-descriptions-item>
          <el-descriptions-item label="生成时间">{{ selectedKey.keyInfo.生成时间 }}</el-descriptions-item>
          <el-descriptions-item label="状态">
            <el-tag :type="getStatusType(selectedKey.keyInfo.作用时间.结束时间)">
              {{ getStatusText(selectedKey.keyInfo.作用时间.结束时间) }}
            </el-tag>
          </el-descriptions-item>
        </el-descriptions>

        <el-divider content-position="left">作用时间</el-divider>
        <el-descriptions :column="2" border>
          <el-descriptions-item label="开始时间">
            <el-tag type="success">{{ selectedKey.keyInfo.作用时间.开始时间 }}</el-tag>
          </el-descriptions-item>
          <el-descriptions-item label="结束时间">
            <el-tag type="warning">{{ selectedKey.keyInfo.作用时间.结束时间 }}</el-tag>
          </el-descriptions-item>
        </el-descriptions>

        <el-divider content-position="left">密钥信息</el-divider>
        <el-descriptions :column="1" border>
          <el-descriptions-item label="完整密钥">
            <div class="key-display">
              <el-input
                :value="selectedKey.keyInfo.密钥"
                readonly
                class="key-input"
              >
                <template v-slot:append>
                  <el-button icon="el-icon-copy-document" @click="copyKeyValue(selectedKey.keyInfo.密钥)">
                    复制
                  </el-button>
                </template>
              </el-input>
            </div>
          </el-descriptions-item>
          <el-descriptions-item label="备注说明">
            {{ selectedKey.keyInfo.备注说明 || '无' }}
          </el-descriptions-item>
        </el-descriptions>

        <el-divider content-position="left">文件信息</el-divider>
        <el-descriptions :column="2" border>
          <el-descriptions-item label="文件大小">{{ formatFileSize(selectedKey.size) }}</el-descriptions-item>
          <el-descriptions-item label="保存时间">{{ selectedKey.saveTime }}</el-descriptions-item>
          <el-descriptions-item label="文件路径" :span="2">{{ selectedKey.file_path }}</el-descriptions-item>
        </el-descriptions>
      </div>
      <span slot="footer" class="dialog-footer">
        <el-button @click="detailDialogVisible = false">关闭</el-button>
        <el-button type="primary" @click="exportKeyInfo">导出密钥信息</el-button>
        <el-button v-if="selectedKey" type="success" @click="copyKeyValue(selectedKey.keyInfo.密钥)">
          复制密钥
        </el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  filters: {
    keyTypeFilter(type) {
      const typeMap = {
        AES: 'success',
        '3DES': 'warning',
        DES: 'info'
      }
      return typeMap[type] || 'primary'
    }
  },
  data() {
    return {
      list: [],
      listLoading: true,
      detailDialogVisible: false,
      selectedKey: null
    }
  },
  created() {
    this.fetchData()
  },
  methods: {
    async fetchData() {
      this.listLoading = true
      try {
        const response = await axios.get('http://localhost:5000/api/list-saved-keys')

        if (response.data.success) {
          // 转换数据格式以匹配表格显示
          this.list = response.data.keys.map(key => ({
            ...key,
            // 从密钥信息中提取显示字段
            algorithm: key.algorithm,
            key_length: key.key_length,
            quantization: '10bit', // 先设置默认值，后续从详细信息中获取
            generation_time: key.generation_time,
            creation_time: key.creation_time,
            description: key.description,
            // 需要从详细信息中获取结束时间，先设置为未知
            end_time: '加载中...'
          }))

          // 异步获取每个密钥的详细信息以获取结束时间和量化位数
          this.fetchDetailedInfo()

          console.log('成功获取密钥历史:', this.list.length, '条记录')
        } else {
          this.$message.error(response.data.message || '获取密钥历史失败')
          this.list = []
        }
      } catch (error) {
        console.error('获取密钥历史失败:', error)
        this.$message.error('网络错误，请检查服务器连接')
        this.list = []
      } finally {
        this.listLoading = false
      }
    },

    deleteKey(row) {
      this.$confirm(`确定要删除密钥文件 "${row.filename}" 吗？删除后将无法恢复。`, '警告', {
        confirmButtonText: '确定删除',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(async() => {
        try {
          const response = await axios.delete(`http://localhost:5000/api/delete-key-info/${row.filename}`)

          if (response.data.success) {
            // 从列表中移除该项
            const index = this.list.findIndex(item => item.filename === row.filename)
            if (index > -1) {
              this.list.splice(index, 1)
            }

            this.$message({
              type: 'success',
              message: '密钥文件已删除'
            })
          } else {
            this.$message.error(response.data.message || '删除失败')
          }
        } catch (error) {
          console.error('删除密钥文件失败:', error)
          this.$message.error('删除失败，请检查网络连接')
        }
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        })
      })
    },

    extractQuantization(filename) {
      // 从文件名中提取量化位数信息
      const match = filename.match(/(\d+bit)/i)
      return match ? match[1] : '10bit'
    },

    async fetchDetailedInfo() {
      // 并行获取所有密钥的详细信息以获取结束时间和量化位数
      try {
        const promises = this.list.map(async(item) => {
          try {
            const response = await axios.get(`http://localhost:5000/api/get-key-info/${item.filename}`)
            if (response.data.success) {
              const keyInfo = response.data.data.密钥信息
              return {
                filename: item.filename,
                endTime: keyInfo.作用时间.结束时间,
                quantization: keyInfo.量化位数 || '10bit'
              }
            }
          } catch (error) {
            console.warn(`获取 ${item.filename} 的详细信息失败:`, error)
            return null
          }
        })

        const results = await Promise.all(promises)

        // 更新列表中的結束时间和量化位数
        results.forEach(result => {
          if (result) {
            const item = this.list.find(item => item.filename === result.filename)
            if (item) {
              item.end_time = result.endTime
              item.quantization = result.quantization
            }
          }
        })

        // 设置未获取到的项目为默认值
        this.list.forEach(item => {
          if (item.end_time === '加载中...') {
            item.end_time = '未知'
          }
          if (!item.quantization || item.quantization === '加载中...') {
            item.quantization = '10bit'
          }
        })
      } catch (error) {
        console.error('批量获取详细信息失败:', error)
      }
    },

    getStatusType(endTime) {
      if (!endTime || endTime === '未知' || endTime === '加载中...') {
        return 'info'
      }

      try {
        const endDate = new Date(endTime)
        const currentDate = new Date()
        return currentDate <= endDate ? 'success' : 'danger'
      } catch (error) {
        return 'info'
      }
    },

    getStatusText(endTime) {
      if (!endTime || endTime === '未知' || endTime === '加载中...') {
        return '未知'
      }

      try {
        const endDate = new Date(endTime)
        const currentDate = new Date()
        return currentDate <= endDate ? '未过期' : '已过期'
      } catch (error) {
        return '未知'
      }
    },

    async viewDetails(row) {
      try {
        // 获取完整的密钥信息
        const response = await axios.get(`http://localhost:5000/api/get-key-info/${row.filename}`)

        if (response.data.success) {
          this.selectedKey = {
            ...row,
            keyInfo: response.data.data.密钥信息,
            saveTime: response.data.data.保存时间,
            fileDescription: response.data.data.文件说明
          }
          this.detailDialogVisible = true
        } else {
          this.$message.error('获取密钥详情失败')
        }
      } catch (error) {
        console.error('获取密钥详情失败:', error)
        this.$message.error('获取密钥详情失败')
      }
    },

    async copyKey(row) {
      try {
        // 获取完整密钥信息用于复制
        const response = await axios.get(`http://localhost:5000/api/get-key-info/${row.filename}`)

        if (response.data.success) {
          const keyValue = response.data.data.密钥信息.密钥
          this.copyKeyValue(keyValue)
        } else {
          this.$message.error('获取密钥失败')
        }
      } catch (error) {
        console.error('获取密钥失败:', error)
        this.$message.error('获取密钥失败')
      }
    },

    copyKeyValue(keyValue) {
      if (keyValue) {
        navigator.clipboard.writeText(keyValue).then(() => {
          this.$message.success('密钥已复制到剪贴板')
        }).catch(() => {
          // 备用方案
          const textArea = document.createElement('textarea')
          textArea.value = keyValue
          document.body.appendChild(textArea)
          textArea.select()
          document.execCommand('copy')
          document.body.removeChild(textArea)
          this.$message.success('密钥已复制到剪贴板')
        })
      } else {
        this.$message.warning('没有可复制的密钥')
      }
    },

    exportKeyInfo() {
      if (!this.selectedKey) return

      // 导出完整的密钥信息
      const exportData = {
        文件信息: {
          文件名: this.selectedKey.filename,
          保存时间: this.selectedKey.saveTime,
          文件大小: this.formatFileSize(this.selectedKey.size)
        },
        密钥信息: this.selectedKey.keyInfo,
        导出时间: new Date().toLocaleString('zh-CN')
      }

      const dataStr = JSON.stringify(exportData, null, 2)
      const dataBlob = new Blob([dataStr], { type: 'application/json' })
      const url = URL.createObjectURL(dataBlob)
      const link = document.createElement('a')
      link.href = url
      link.download = `${this.selectedKey.keyInfo.加密算法}_密钥详情_${new Date().toISOString().slice(0, 10)}.json`
      link.click()
      URL.revokeObjectURL(url)

      this.$message({
        type: 'success',
        message: '密钥信息已导出'
      })
    },

    openKeyDirectory() {
      // 提示用户手动打开目录
      this.$alert('密钥文件保存在：E:\\DelayKeyGeneration\\keysInfo', '密钥目录', {
        confirmButtonText: '确定',
        type: 'info'
      })
    },

    formatFileSize(bytes) {
      if (!bytes) return '0 B'
      const k = 1024
      const sizes = ['B', 'KB', 'MB', 'GB']
      const i = Math.floor(Math.log(bytes) / Math.log(k))
      return parseFloat((bytes / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i]
    }
  }
}
</script>

<style scoped>
.app-container {
  padding: 20px;
}

.filter-container {
  padding-bottom: 10px;
}

.filter-item {
  margin-right: 10px;
}

.link-type {
  color: #409EFF;
  cursor: pointer;
}

.link-type:hover {
  color: #66b1ff;
}

.dialog-footer {
  text-align: right;
}

.key-display {
  margin: 10px 0;
}

.key-input {
  font-family: 'Courier New', monospace;
}

.key-input .el-input__inner {
  font-family: inherit;
  font-size: 13px;
  color: #303133;
  background: #f5f7fa;
  border: 1px solid #dcdfe6;
}

.key-input .el-input-group__append {
  background: #409EFF;
  border-color: #409EFF;
}

.key-input .el-input-group__append .el-button {
  background: transparent;
  border: none;
  color: white;
}

.key-input .el-input-group__append .el-button:hover {
  background: rgba(255, 255, 255, 0.1);
}

.el-descriptions {
  margin-bottom: 20px;
}

.el-tag {
  margin: 2px;
}

/* 响应式设计 */
@media (max-width: 1200px) {
  .el-table-column {
    min-width: 120px;
  }
}
</style>
