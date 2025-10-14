<template>
  <div class="key-request-container">
    <!-- 请求表单 -->
    <div class="request-form">
      <div class="config-section">
        <div class="form-group">
          <label class="form-label">作用时间</label>
          <div class="time-config">
            <div class="start-time">
              <span class="time-label">开始时间</span>
              <el-date-picker
                v-model="form.startDate"
                type="datetime"
                placeholder="选择开始时间"
                format="yyyy-MM-dd HH:mm:ss"
                value-format="yyyy-MM-dd HH:mm:ss"
                style="width: 100%;"
              />
            </div>

            <div class="duration-select">
              <span class="time-label">时间范围</span>
              <el-select v-model="form.duration" style="width: 100%;" @change="onDurationChange">
                <el-option label="一天" value="1day" />
                <el-option label="一周" value="1week" />
                <el-option label="一个月" value="1month" />
                <el-option label="自定义" value="custom" />
              </el-select>
            </div>

            <div class="end-time">
              <span class="time-label">结束时间</span>
              <el-date-picker
                v-model="form.endDate"
                type="datetime"
                placeholder="选择结束时间"
                format="yyyy-MM-dd HH:mm:ss"
                value-format="yyyy-MM-dd HH:mm:ss"
                :disabled="form.duration !== 'custom'"
                style="width: 100%;"
              />
            </div>
          </div>
        </div>

        <div class="form-group">
          <label class="form-label">加密算法</label>
          <el-radio-group v-model="form.algorithm" class="algorithm-group" @change="onAlgorithmChange">
            <el-radio label="AES">AES</el-radio>
            <el-radio label="DES">DES</el-radio>
            <el-radio label="3DES">3DES</el-radio>
          </el-radio-group>
          <div class="algorithm-hint">
            <span v-if="form.algorithm === 'AES'" class="hint-text">
              <i class="el-icon-info" />
              AES：高级加密标准，支持128/192/256位密钥
            </span>
            <span v-if="form.algorithm === 'DES'" class="hint-text">
              <i class="el-icon-info" />
              DES：数据加密标准，使用64位密钥
            </span>
            <span v-if="form.algorithm === '3DES'" class="hint-text">
              <i class="el-icon-info" />
              3DES：三重DES加密，支持112/168位密钥
            </span>
          </div>
        </div>



        <el-row :gutter="16">
          <el-col :span="12">
            <div class="form-group">
              <label class="form-label">量化位数</label>
              <el-select v-model="form.quantization" style="width: 100%;">
                <el-option label="10位" value="10bit" />
              </el-select>
            </div>
          </el-col>
          <el-col :span="12">
            <div class="form-group">
              <label class="form-label">密钥长度</label>
              <el-select v-model="form.keyLength" style="width: 100%;">
                <el-option
                  v-for="option in getKeyLengthOptions()"
                  :key="option.value"
                  :label="option.label"
                  :value="option.value"
                />
              </el-select>
            </div>
          </el-col>
        </el-row>

        <div class="form-group">
          <label class="form-label">目标设备</label>
          <el-radio-group v-model="form.device" class="device-group">
            <el-radio label="raspberry">
              <i class="el-icon-cpu" />
              树莓派
            </el-radio>
            <el-radio label="orange">
              <i class="el-icon-cpu" />
              香橙派
            </el-radio>
          </el-radio-group>
        </div>

        <div class="form-group">
          <label class="form-label">备注说明</label>
          <el-input
            v-model="form.description"
            type="textarea"
            :rows="3"
            placeholder="请输入密钥用途或其他说明信息"
            maxlength="200"
            show-word-limit
          />
        </div>

        <!-- 操作按钮 -->
        <div class="action-buttons">
          <el-button type="primary" size="large" :loading="submitting" @click="onSubmit">
            <i class="el-icon-check" />
            提交申请
          </el-button>
          <el-button size="large" @click="onReset">
            <i class="el-icon-refresh" />
            重置表单
          </el-button>
        </div>
      </div>
    </div>

    <!-- 编码结果对话框 -->
    <el-dialog
      title="密钥生成结果（纠错+拼接+隐私放大）"
      :visible.sync="resultDialogVisible"
      width="90%"
      :close-on-click-modal="false"
    >
      <!-- 只有当keyResult存在时才渲染内容 -->
      <div v-if="keyResult">
        <!-- 样本数据详情 -->
        <div class="data-block">
          <h4>样本数据详情（30）：</h4>
          <div class="detailed-data-table">
            <div class="table-header">
              <div class="col-index">序号</div>
              <div class="col-client-wide">Client1</div>
              <div class="col-client-wide">Client2</div>
              <div class="col-correction">纠错状态</div>
              <div class="col-final">最终编码</div>
            </div>
            <div class="table-body">
              <div v-for="(item, index) in getDisplayData()" :key="index" class="table-row">
                <div class="col-index">{{ item.index }}</div>

                <!-- Client1 信息 -->
                <div class="col-client-wide">
                  <div v-if="item.client1" class="client-info">
                    <div class="level">级别: {{ item.client1.level || '未知' }}</div>
                    <div class="delay">时延: {{ item.client1.delay_ms || 0 }}ms</div>
                    <div class="hash-info">哈希: <span class="hash-value">{{ item.client1.hash || '未知' }}</span></div>
                    <div class="gray-code">格雷码: {{ item.client1.gray_code || '未知' }}</div>
                  </div>
                </div>

                <!-- Client2 信息 -->
                <div class="col-client-wide">
                  <div v-if="item.client2" class="client-info">
                    <div class="level">级别: {{ item.client2.level || '未知' }}</div>
                    <div class="delay">时延: {{ item.client2.delay_ms || 0 }}ms</div>
                    <div class="hash-info">哈希: <span class="hash-value">{{ item.client2.hash || '未知' }}</span></div>
                    <div class="gray-code original">原始: {{ item.client2.gray_code || '未知' }}</div>
                    <div v-if="item.client2.needs_correction" class="gray-code corrected">
                      纠错: {{ item.client2.corrected_code || '未知' }}
                    </div>
                  </div>
                </div>

                <!-- 纠错状态 -->
                <div class="col-correction">
                  <div class="correction-status">
                    <div class="hash-match" :class="{ 'match': item.hash_match, 'mismatch': !item.hash_match }">
                      <i :class="item.hash_match ? 'el-icon-success' : 'el-icon-warning'" />
                      {{ item.hash_match ? '哈希匹配' : '哈希不匹配' }}
                    </div>
                    <div class="correction-needed" :class="{ 'needed': item.client2 && item.client2.needs_correction, 'no-need': !item.client2 || !item.client2.needs_correction }">
                      <i :class="(item.client2 && item.client2.needs_correction) ? 'el-icon-edit' : 'el-icon-check'" />
                      {{ (item.client2 && item.client2.needs_correction) ? '需要纠错' : '无需纠错' }}
                    </div>
                  </div>
                </div>

                <!-- 最终编码 -->
                <div class="col-final">
                  <div class="final-code">
                    <div class="code-title">最终编码:</div>
                    <div class="code-value">{{ item.final_code || '未知' }}</div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 最终密钥展示 -->
        <div v-if="keyResult.final_key" class="final-key-block">
          <h4>生成的最终密钥：</h4>
          <div class="key-display">
            <div class="key-info">
              <div class="key-param">
                <span class="key-label">适用算法:</span>
                <span class="key-value">{{ keyResult.final_key.algorithm || '未知' }}</span>
              </div>
              <div class="key-param">
                <span class="key-label">密钥长度:</span>
                <span class="key-value">{{ keyResult.final_key.key_length_bits || 0 }} 位 ({{ keyResult.final_key.key_length_bytes || 0 }} 字节)</span>
              </div>
              <div class="key-param">
                <span class="key-label">量化位数:</span>
                <span class="key-value">{{ form.quantization.replace('bit', '位') }}</span>
              </div>
            </div>

            <div class="key-content">
              <div class="key-full">
                <span class="key-label">完整密钥 (十六进制):</span>
                <el-input
                  :value="keyResult.final_key.hex_key || ''"
                  readonly
                  class="key-input"
                >
                  <el-button slot="append" icon="el-icon-copy-document" @click="copyKey">复制</el-button>
                </el-input>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 当keyResult为null时显示的内容 -->
      <div v-else class="no-data">
        <p>暂无数据显示</p>
      </div>

      <span slot="footer" class="dialog-footer">
        <el-button @click="resultDialogVisible = false">关 闭</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'KeyRequestForm',
  data() {
    return {
      submitting: false,
      resultDialogVisible: false,
      keyResult: null,
      form: {
        startDate: this.getCurrentDateTime(),
        endDate: '',
        duration: '1day',
        algorithm: 'AES',
        quantization: '10bit',
        device: 'raspberry',
        keyLength: '128',
        description: ''
      }
    }
  },
  mounted() {
    this.setEndDateByDuration('1day')
    this.onAlgorithmChange(this.form.algorithm)
  },
  methods: {
    // 获取当前时间字符串
    getCurrentDateTime() {
      const now = new Date()
      const year = now.getFullYear()
      const month = String(now.getMonth() + 1).padStart(2, '0')
      const day = String(now.getDate()).padStart(2, '0')
      const hours = String(now.getHours()).padStart(2, '0')
      const minutes = String(now.getMinutes()).padStart(2, '0')
      const seconds = String(now.getSeconds()).padStart(2, '0')
      return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
    },

    // 算法改变时的处理
    onAlgorithmChange(algorithm) {
      const defaultLengths = {
        'AES': '128',
        'DES': '64',
        '3DES': '168'
      }
      this.form.keyLength = defaultLengths[algorithm] || '128'
    },

    // 获取密钥长度选项
    getKeyLengthOptions() {
      const options = {
        'AES': [
          { label: '128位', value: '128' },
          { label: '192位', value: '192' },
          { label: '256位', value: '256' }
        ],
        'DES': [
          { label: '64位', value: '64' }
        ],
        '3DES': [
          { label: '112位', value: '112' },
          { label: '168位', value: '168' }
        ]
      }
      return options[this.form.algorithm] || options['AES']
    },

    // 根据时间范围设置结束时间
    onDurationChange(duration) {
      if (duration !== 'custom') {
        this.setEndDateByDuration(duration)
      }
    },

    setEndDateByDuration(duration) {
      if (duration === 'custom') {
        return
      }

      const startDate = new Date(this.form.startDate)
      const endDate = new Date(startDate)

      switch (duration) {
        case '1day':
          endDate.setDate(startDate.getDate() + 1)
          break
        case '1week':
          endDate.setDate(startDate.getDate() + 7)
          break
        case '1month':
          endDate.setMonth(startDate.getMonth() + 1)
          break
      }

      this.form.endDate = this.formatDateTime(endDate)
    },

    // 格式化日期时间
    formatDateTime(date) {
      const year = date.getFullYear()
      const month = String(date.getMonth() + 1).padStart(2, '0')
      const day = String(date.getDate()).padStart(2, '0')
      const hours = String(date.getHours()).padStart(2, '0')
      const minutes = String(date.getMinutes()).padStart(2, '0')
      const seconds = String(date.getSeconds()).padStart(2, '0')
      return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`
    },

    // 表单验证
    validateForm() {
      if (!this.form.startDate || !this.form.endDate) {
        this.$message.error('请设置完整的作用时间')
        return false
      }

      if (new Date(this.form.endDate) <= new Date(this.form.startDate)) {
        this.$message.error('结束时间必须晚于开始时间')
        return false
      }

      if (!this.form.algorithm) {
        this.$message.error('请选择一种加密算法')
        return false
      }

      return true
    },

    // 提交申请
    async onSubmit() {
      if (!this.validateForm()) {
        return
      }

      this.submitting = true

      const submitData = {
        startTime: this.form.startDate,
        endTime: this.form.endDate,
        duration: this.form.duration,
        algorithm: this.form.algorithm,
        quantization: this.form.quantization,
        targetDevice: this.form.device,
        keyLength: this.form.keyLength,
        description: this.form.description,
        requestTime: this.getCurrentDateTime()
      }

      console.log('提交密钥申请:', submitData)

      try {
        const response = await axios.post('http://localhost:5000/api/generate-key', submitData)

        if (response.data.success) {
          this.keyResult = response.data.data
          this.resultDialogVisible = true
          this.$message.success('密钥生成成功！')

          // 自动保存密钥信息到本地
          this.saveKeyInfo()
        } else {
          this.$message.error(response.data.message || '密钥生成失败')
        }
      } catch (error) {
        console.error('密钥生成错误:', error)

        if (error.response && error.response.data) {
          this.$message.error(error.response.data.message || '密钥生成失败')
        } else {
          this.$message.error('网络错误，请检查服务器连接')
        }
      } finally {
        this.submitting = false
      }
    },

    // 重置表单
    onReset() {
      this.form = {
        startDate: this.getCurrentDateTime(),
        endDate: '',
        duration: '1day',
        algorithm: 'AES',
        quantization: '10bit',
        device: 'raspberry',
        keyLength: '128',
        description: ''
      }
      this.setEndDateByDuration('1day')
      this.$message.success('表单已重置')
    },

    // 获取展示数据 - 添加安全检查
    getDisplayData() {
      if (this.keyResult && this.keyResult.display_data && Array.isArray(this.keyResult.display_data)) {
        return this.keyResult.display_data
      }
      return []
    },

    // 复制密钥到剪贴板
    copyKey() {
      if (this.keyResult && this.keyResult.final_key && this.keyResult.final_key.hex_key) {
        navigator.clipboard.writeText(this.keyResult.final_key.hex_key).then(() => {
          this.$message.success('密钥已复制到剪贴板')
        }).catch(() => {
          // 备用方案
          const textArea = document.createElement('textarea')
          textArea.value = this.keyResult.final_key.hex_key
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

    // 保存密钥信息到本地
    async saveKeyInfo() {
      if (!this.keyResult || !this.keyResult.final_key) {
        return
      }

      try {
        const keyInfo = {
          作用时间: {
            开始时间: this.form.startDate,
            结束时间: this.form.endDate
          },
          加密算法: this.form.algorithm,
          量化位数: this.form.quantization.replace('bit', '位'),
          密钥长度: `${this.keyResult.final_key.key_length_bits}位`,
          密钥: this.keyResult.final_key.hex_key,
          生成时间: this.keyResult.generation_time,
          备注说明: this.form.description || '无'
        }

        const response = await axios.post('http://localhost:5000/api/save-key-info', {
          keyInfo: keyInfo,
          basePath: 'E:\\DelayKeyGeneration\\keysInfo'
        })

        if (response.data.success) {
          console.log('密钥信息已自动保存到本地')
        } else {
          console.warn('密钥信息保存失败:', response.data.message)
        }
      } catch (error) {
        console.error('保存密钥信息时出错:', error)
      }
    }
  }
}
</script>

<style scoped>
.key-request-container {
  padding: 24px;
  background-color: #f5f7fa;
  min-height: calc(100vh - 84px);
}

.request-form {
  background: white;
  border-radius: 12px;
  padding: 32px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.08);
}

.config-section {
  max-width: 900px;
  margin: 0 auto;
}

.form-group {
  margin-bottom: 24px;
}

.form-label {
  display: block;
  margin-bottom: 8px;
  font-weight: 600;
  color: #606266;
  font-size: 14px;
}

.time-config {
  display: grid;
  grid-template-columns: 1fr 120px 1fr;
  gap: 16px;
  align-items: end;
}

.time-label {
  display: block;
  font-size: 12px;
  color: #909399;
  margin-bottom: 4px;
}

.algorithm-group {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}

.algorithm-group .el-radio {
  margin: 0;
  padding: 8px 16px;
  border: 1px solid #dcdfe6;
  border-radius: 6px;
  transition: all 0.3s;
}

.algorithm-group .el-radio.is-checked {
  border-color: #409eff;
  background-color: #ecf5ff;
}



.device-group {
  display: flex;
  gap: 16px;
}

.device-group .el-radio {
  margin: 0;
  padding: 12px 16px;
  border: 1px solid #dcdfe6;
  border-radius: 8px;
  transition: all 0.3s;
  flex: 1;
  text-align: center;
}

.device-group .el-radio.is-checked {
  border-color: #409eff;
  background-color: #ecf5ff;
}

.device-group .el-radio i {
  margin-right: 4px;
  font-size: 16px;
}

.hint-text {
  font-size: 13px;
  color: #909399;
  display: flex;
  align-items: center;
}

.hint-text i {
  margin-right: 4px;
  color: #409eff;
}

.action-buttons {
  margin-top: 32px;
  text-align: center;
  padding-top: 24px;
  border-top: 1px solid #ebeef5;
}

.action-buttons .el-button {
  min-width: 120px;
  margin: 0 8px;
}

/* 对话框样式 */
.result-info {
  background-color: #f5f7fa;
  padding: 16px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.info-label {
  font-weight: 600;
  color: #606266;
}

.info-text {
  color: #303133;
}

/* 纠错统计样式 */
.correction-stats {
  background: #fff;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
}

.correction-stats h4 {
  margin: 0 0 16px 0;
  font-size: 16px;
  color: #303133;
  font-weight: 600;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 16px;
}

.stat-item {
  text-align: center;
  padding: 12px;
  background: #f8f9fa;
  border-radius: 6px;
}

.stat-value {
  font-size: 20px;
  font-weight: bold;
  color: #409eff;
  margin-bottom: 4px;
}

.stat-label {
  font-size: 12px;
  color: #909399;
}

.stat-note {
  font-size: 10px;
  color: #909399;
  margin-top: 2px;
  line-height: 1.2;
}

.stats-explanation {
  margin-top: 16px;
}

.stats-explanation p {
  margin: 4px 0;
  font-size: 13px;
  line-height: 1.4;
}

.data-block {
  margin-bottom: 20px;
}

.data-block h4 {
  margin: 0 0 10px 0;
  font-size: 16px;
  color: #303133;
  font-weight: 600;
}

/* 详细数据表格样式 */
.detailed-data-table {
  border: 1px solid #e4e7ed;
  border-radius: 6px;
  overflow: hidden;
  background: white;
}

.table-header {
  display: grid;
  grid-template-columns: 60px 1fr 1fr 180px 150px;
  background: #f5f7fa;
  border-bottom: 1px solid #e4e7ed;
  font-weight: 600;
  color: #606266;
}

.table-header > div {
  padding: 12px;
  text-align: center;
  border-right: 1px solid #e4e7ed;
}

.table-header > div:last-child {
  border-right: none;
}

.table-body {
  max-height: 600px;
  overflow-y: auto;
}

.table-row {
  display: grid;
  grid-template-columns: 60px 1fr 1fr 180px 150px;
  border-bottom: 1px solid #f0f0f0;
  min-height: 120px;
}

.table-row:last-child {
  border-bottom: none;
}

.table-row:hover {
  background-color: #fafafa;
}

.col-index {
  padding: 12px;
  text-align: center;
  font-weight: 600;
  color: #409eff;
  border-right: 1px solid #f0f0f0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.col-client-wide {
  padding: 8px 12px;
  border-right: 1px solid #f0f0f0;
}

.col-correction {
  padding: 8px 12px;
  border-right: 1px solid #f0f0f0;
}

.col-final {
  padding: 8px 12px;
  border-right: none;
}

.client-info {
  font-size: 12px;
}

.client-info .level {
  color: #409eff;
  font-weight: 600;
  margin-bottom: 4px;
}

.client-info .delay {
  color: #606266;
  margin-bottom: 4px;
}

.client-info .hash-info {
  color: #909399;
  margin-bottom: 4px;
  font-size: 11px;
}

.hash-value {
  font-family: 'Courier New', monospace;
  background: #f0f0f0;
  padding: 1px 4px;
  border-radius: 2px;
}

.client-info .gray-code {
  font-family: 'Courier New', monospace;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 11px;
  margin-bottom: 2px;
}

.gray-code.original {
  color: #e6a23c;
  background: #fdf6ec;
}

.gray-code.corrected {
  color: #67c23a;
  background: #f0f9ff;
}

.correction-status {
  font-size: 12px;
}

.hash-match, .correction-needed {
  margin-bottom: 6px;
  padding: 2px 6px;
  border-radius: 4px;
  display: flex;
  align-items: center;
}

.hash-match.match {
  color: #67c23a;
  background: #f0f9ff;
}

.hash-match.mismatch {
  color: #f56c6c;
  background: #fef0f0;
}

.correction-needed.needed {
  color: #e6a23c;
  background: #fdf6ec;
}

.correction-needed.no-need {
  color: #67c23a;
  background: #f0f9ff;
}

.correction-needed i, .hash-match i {
  margin-right: 2px;
  font-size: 10px;
}

/* 最终编码列样式 */
.final-code {
  font-size: 12px;
}

.final-code .code-title {
  color: #67c23a;
  font-weight: 600;
  margin-bottom: 4px;
}

.final-code .code-value {
  font-family: 'Courier New', monospace;
  color: #303133;
  background: #f0f9ff;
  padding: 4px 6px;
  border-radius: 4px;
  font-size: 11px;
  word-break: break-all;
  line-height: 1.3;
}

/* 最终密钥区块样式 */
.final-key-block {
  background: #fff;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
  padding: 20px;
  margin: 20px 0;
  border-left: 4px solid #67c23a;
}

.final-key-block h4 {
  margin: 0 0 16px 0;
  font-size: 16px;
  color: #303133;
  font-weight: 600;
  border-bottom: 2px solid #67c23a;
  padding-bottom: 8px;
}

.key-display {
  background: #f0f9ff;
  border-radius: 6px;
  padding: 16px;
}

.key-info {
  display: flex;
  gap: 24px;
  margin-bottom: 16px;
  padding-bottom: 12px;
  border-bottom: 1px solid #e1f5fe;
  flex-wrap: wrap;
}

.key-param {
  display: flex;
  align-items: center;
  gap: 8px;
}

.key-label {
  font-size: 13px;
  color: #606266;
  font-weight: 500;
}

.key-value {
  font-size: 13px;
  color: #303133;
  font-weight: 600;
}

.key-content {
  space-y: 12px;
}

.key-full {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.key-input {
  font-family: 'Courier New', monospace;
}

.key-input .el-input__inner {
  font-family: inherit;
  font-size: 13px;
  color: #303133;
  background: white;
  border: 2px solid #67c23a;
}

.key-input .el-input-group__append {
  background: #67c23a;
  border-color: #67c23a;
}

.key-input .el-input-group__append .el-button {
  background: transparent;
  border: none;
  color: white;
}

.key-input .el-input-group__append .el-button:hover {
  background: #20a0ff;
}

/* 对话框底部按钮 */
.dialog-footer .el-button {
  margin-left: 8px;
}

/* 响应式设计 */
@media (max-width: 1200px) {
  .time-config {
    grid-template-columns: 1fr;
    gap: 12px;
  }

  .table-header,
  .table-row {
    grid-template-columns: 50px 1fr 1fr 140px 120px;
  }

  .key-info {
    flex-direction: column;
    gap: 8px;
  }
}

@media (max-width: 768px) {
  .key-request-container {
    padding: 16px;
  }

  .request-form {
    padding: 20px;
  }

  .table-header,
  .table-row {
    grid-template-columns: 1fr;
    grid-template-rows: auto auto auto auto auto;
  }
}

/* 滚动条样式 */
.table-body::-webkit-scrollbar {
  width: 6px;
}

.table-body::-webkit-scrollbar-track {
  background: #f5f7fa;
  border-radius: 3px;
}

.table-body::-webkit-scrollbar-thumb {
  background: #c0c4cc;
  border-radius: 3px;
}

.table-body::-webkit-scrollbar-thumb:hover {
  background: #a6a9ad;
}
</style>
