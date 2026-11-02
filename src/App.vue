<template>
  <div class="container">
    <el-card class="box-card">
      <template #header>
        <div class="card-header">
          <h2>西南联交所 - 投标文件递交系统 (Demo)</h2>
        </div>
      </template>

      <div class="content">
        <el-alert
          title="安全提示：递交前需校验电脑硬件环境，防止串标围标。"
          type="info"
          :closable="false"
          show-icon
          style="margin-bottom: 20px;"
        />

        <div class="action-area">
          <el-button 
            type="primary" 
            size="large" 
            :loading="isLoading" 
            :disabled="isLoading"
            @click="handleSubmitBid"
          >
            {{ isLoading ? '正在校验环境...' : '递交投标文件' }}
          </el-button>
        </div>

        <el-progress 
          v-if="uploadProgress > 0" 
          :percentage="uploadProgress" 
          :status="uploadProgress === 100 ? 'success' : ''"
          style="margin-top: 20px; margin-bottom: 20px;"
        />

        <!-- 采集信息回显面板 -->
        <el-card v-if="hardwareInfo" class="result-card" shadow="never">
          <template #header>
            <div class="result-header">
              <el-icon color="#67C23A" :size="20"><CircleCheck /></el-icon>
              <span style="margin-left: 8px; font-weight: bold; color: #67C23A;">硬件环境采集成功</span>
            </div>
          </template>
          
          <el-descriptions :column="1" border size="small">
            <el-descriptions-item label="操作系统">
              <el-tag>{{ currentOSName }}</el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="内网 IP">
              <el-tag type="info">{{ hardwareInfo.inner_ip || '未获取到' }}</el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="外网 IP">
              <el-tag type="info">{{ hardwareInfo.outer_ip || '未获取到' }}</el-tag>
            </el-descriptions-item>
            <el-descriptions-item label="CPU 编号 (Hash)">
              <span class="hash-text">{{ formatHash(hardwareInfo.cpu_id)}}</span>
            </el-descriptions-item>
            <el-descriptions-item label="主板序列号 (Hash)">
              <span class="hash-text">{{ formatHash(hardwareInfo.board_serial)}}</span>
            </el-descriptions-item>
            <el-descriptions-item label="MAC 地址 (Hash)">
              <div v-if="hardwareInfo.mac_addresses && hardwareInfo.mac_addresses.length > 0">
                <div v-for="(mac, index) in hardwareInfo.mac_addresses" :key="index" class="hash-text">
                  {{ formatHash(mac) }}
                </div>
              </div>
              <span v-else class="hash-text">未获取到</span>
            </el-descriptions-item>
            <el-descriptions-item label="硬盘序列号 (Hash)">
              <div v-if="hardwareInfo.disk_serials && hardwareInfo.disk_serials.length > 0">
                <div v-for="(disk, index) in hardwareInfo.disk_serials" :key="index" class="hash-text">
                  {{ formatHash(disk) }}
                </div>
              </div>
              <span v-else class="hash-text">未获取到</span>
            </el-descriptions-item>
          </el-descriptions>
        </el-card>

      </div>
    </el-card>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { CircleCheck } from '@element-plus/icons-vue'

const isLoading = ref(false)
const uploadProgress = ref(0)
const hardwareInfo = ref(null)
let isAlertShowing = false
let isProcessing = false 

const LOCAL_GO_URL = 'http://127.0.0.1:38080'

// 【新增】识别当前操作系统
const currentOS = ref('windows')
const currentOSName = ref('Windows')

const getOS = () => {
  const userAgent = navigator.userAgent.toLowerCase()
  const platform = navigator.platform.toLowerCase()
  
  if (platform.includes('mac') || userAgent.includes('mac')) {
    currentOS.value = 'mac'
    currentOSName.value = 'macOS'
    return 'mac'
  }
  currentOS.value = 'windows'
  currentOSName.value = 'Windows'
  return 'windows'
}

// 初始化时识别系统
getOS()

const handleSubmitBid = async () => {
  if (isProcessing || isLoading.value) return
  
  isProcessing = true 
  isLoading.value = true
  uploadProgress.value = 0
  hardwareInfo.value = null

  // 1. 先探活
  const pingSuccess = await checkLocalAppRunning()

  if (!pingSuccess) {
    isLoading.value = false
    isProcessing = false
    
    if (!isAlertShowing) {
      isAlertShowing = true
      
      // 【修改】根据系统生成不同的提示语和下载链接
      const isMac = currentOS.value === 'mac'
      const downloadUrl = isMac ? '/安全助手_Mac版.zip' : '/安全助手_windows.exe'
      const tipText = isMac 
        ? '未检测到【安全助手】正在运行。请先下载 Mac 版安全助手并运行后（需在终端赋予执行权限），再递交投标文件。'
        : '未检测到【安全助手】正在运行。请先下载并运行安全助手后，再递交投标文件。'

      ElMessageBox.alert(
        tipText,
        '环境检测失败',
        { confirmButtonText: '下载安全助手', type: 'warning', showClose: false }
      ).then(() => {
        // 引导用户下载对应系统的文件
        window.location.href = downloadUrl
      }).catch(() => {}).finally(() => {
        isAlertShowing = false
      })
    }
    return 
  }

  // 2. 探活成功，才进入 try-catch 进行采集
  try {
    ElMessage.info('安全助手已连接，正在采集环境信息...')
    
    const payload = { projectId: 'PROJ_2026_001', supplierId: 'SUPP_8888' }
    const collectRes = await fetch(`${LOCAL_GO_URL}/collect`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    })

    if (!collectRes.ok) throw new Error('本地程序采集失败')
    
    const collectData = await collectRes.json()
    if (collectData.status === 'success' && collectData.hardware) {
      hardwareInfo.value = collectData.hardware 
    }

    ElMessage.success('环境校验通过，开始上传文件...')
    simulateFileUpload()

  } catch (error) {
    console.error("采集过程异常:", error)
    ElMessage.error('硬件信息采集失败，请检查安全助手状态！')
    isLoading.value = false
    isProcessing = false
  }
}

const checkLocalAppRunning = async () => {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), 1500)
  try {
    const pingRes = await fetch(`${LOCAL_GO_URL}/ping`, {
      method: 'GET',
      mode: 'cors',
      cache: 'no-store',
      signal: controller.signal
    })
    return pingRes.ok
  } catch (e) {
    return false
  } finally {
    clearTimeout(timeoutId)
  }
}

const formatHash = (hash) => {
  if (!hash) return '未获取到';
  return hash.substring(0, 8) + '...' + hash.substring(hash.length - 8);
}

const simulateFileUpload = () => {
  let progress = 0
  const timer = setInterval(() => {
    progress += 10
    uploadProgress.value = progress
    if (progress >= 100) {
      clearInterval(timer)
      isLoading.value = false
      isProcessing = false 
      ElMessage.success('投标文件递交成功！') 
    }
  }, 300)
}
</script>

<style scoped>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #f0f2f5;
  padding: 20px;
}

.box-card {
  width: 700px;
}

.card-header {
  text-align: center;
  color: #333;
}

.content {
  padding: 10px 20px;
}

.action-area {
  display: flex;
  justify-content: center;
  margin-top: 10px;
}

.result-card {
  margin-top: 20px;
  background-color: #f8f9fa;
  border: 1px solid #e4e7ed;
}

.result-header {
  display: flex;
  align-items: center;
}

.hash-text {
  font-family: monospace;
  word-break: break-all;
  color: #606266;
  font-size: 12px;
}
</style>