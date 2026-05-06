<template>
  <div class="page">
    <div class="sidebar">

      <!-- ── Mode selector ──────────────────────────────────────────── -->
      <div class="section">
        <div class="section-title">配置模式</div>
        <div class="mode-grid">
          <button class="mode-btn" :class="{ on: mode === 'node' }" @click="mode = 'node'">
            <span class="mode-icon">🔗</span>
            <span class="mode-name">单节点</span>
            <span class="mode-desc">粘贴分享链接启动</span>
          </button>
          <button class="mode-btn" :class="{ on: mode === 'subscription' }" @click="mode = 'subscription'">
            <span class="mode-icon">📦</span>
            <span class="mode-name">订阅模式</span>
            <span class="mode-desc">导入订阅、向导配置</span>
          </button>
          <button class="mode-btn" :class="{ on: mode === 'upload' }" @click="mode = 'upload'">
            <span class="mode-icon">📄</span>
            <span class="mode-name">上传配置</span>
            <span class="mode-desc">直接使用 JSON 配置</span>
          </button>
        </div>
      </div>

      <!-- NODE MODE -->
      <template v-if="mode === 'node'">
        <div class="section">
          <div class="section-title-row">
            <span class="section-title">节点列表</span>
            <button class="icon-btn secondary" @click="showImport = true">+ 导入</button>
          </div>
          <div v-if="nodesStore.nodes.length === 0" class="empty-tip">
            还没有节点，点击「导入」粘贴分享链接
          </div>
          <div v-else class="node-list">
            <div v-for="n in nodesStore.nodes" :key="n.id"
              class="node-row" :class="{ active: selectedNodeId === n.id }"
              @click="selectedNodeId = n.id">
              <span class="proto-badge" :class="'p-' + n.type">{{ n.type }}</span>
              <div class="node-meta">
                <span class="node-name">{{ n.name }}</span>
                <span class="node-addr">{{ n.server }}:{{ n.port }}</span>
              </div>
              <button class="del-btn" :disabled="isRunning" @click.stop="deleteNode(n.id)">✕</button>
            </div>
          </div>
        </div>
        <div class="section">
          <div class="section-title">代理模式</div>
          <div class="proxy-grid">
            <button v-for="pm in proxyModes" :key="pm.v"
              class="grid-btn" :class="{ on: proxyMode === pm.v }"
              :disabled="isRunning" @click="proxyMode = pm.v">
              <span class="grid-icon">{{ pm.icon }}</span>
              <span class="grid-name">{{ pm.name }}</span>
              <span class="grid-desc">{{ pm.desc }}</span>
            </button>
          </div>
        </div>
        <div class="section">
          <div class="section-title">路由模式</div>
          <div class="route-grid">
            <button v-for="rm in routeModes" :key="rm.v"
              class="grid-btn" :class="{ on: routeMode === rm.v }"
              :disabled="isRunning" @click="routeMode = rm.v">
              <span class="grid-icon">{{ rm.icon }}</span>
              <span class="grid-name">{{ rm.name }}</span>
              <span class="grid-desc">{{ rm.desc }}</span>
            </button>
          </div>
        </div>
        <div class="section">
          <div class="toggle-group">
            <label class="toggle-row" :class="{ disabled: isRunning }">
              <div class="toggle-track" :class="{ on: lanProxy }"
                @click="!isRunning && (lanProxy = !lanProxy)">
                <div class="toggle-thumb"></div>
              </div>
              <div class="toggle-labels">
                <span class="toggle-name">局域网代理</span>
                <span class="toggle-hint">监听 :: 允许局域网设备连接</span>
              </div>
            </label>
            <label class="toggle-row" :class="{ disabled: isRunning }">
              <div class="toggle-track" :class="{ on: ipv6 }"
                @click="!isRunning && (ipv6 = !ipv6)">
                <div class="toggle-thumb"></div>
              </div>
              <div class="toggle-labels">
                <span class="toggle-name">IPv6 支持</span>
                <span class="toggle-hint">在路由规则中启用 IPv6</span>
              </div>
            </label>
            <label class="toggle-row" :class="{ disabled: isRunning }">
              <div class="toggle-track" :class="{ on: blockAds }"
                @click="!isRunning && (blockAds = !blockAds)">
                <div class="toggle-thumb"></div>
              </div>
              <div class="toggle-labels">
                <span class="toggle-name">广告拦截</span>
                <span class="toggle-hint">使用 Category-Ads 规则集</span>
              </div>
            </label>
          </div>
        </div>
      </template>

      <!-- SUBSCRIPTION MODE -->
      <template v-if="mode === 'subscription'">
        <div class="section">
          <div class="section-title-row">
            <span class="section-title">订阅列表</span>
            <button class="icon-btn secondary" @click="openWizard(null)">+ 新建订阅</button>
          </div>
          <div v-if="subsStore.subs.length === 0" class="empty-tip">
            点击「新建订阅」填写订阅 URL，向导将引导你完成配置
          </div>
          <div class="sub-list">
            <div v-for="sub in subsStore.subs" :key="sub.id"
              class="sub-card" :class="{ active: selectedSubId === sub.id }">
              <div class="sub-header" :class="{ selected: selectedSubId === sub.id }"
                @click="selectedSubId = selectedSubId === sub.id ? null : sub.id">
                <span class="sub-icon">📦</span>
                <div class="sub-info">
                  <div class="sub-name">{{ sub.name }}</div>
                  <div class="sub-meta" :class="{ err: sub.error }">
                    <template v-if="sub.error">⚠ {{ sub.error }}</template>
                    <template v-else-if="sub.nodeCount">
                      {{ sub.nodeCount }} 个节点 · 更新于 {{ fmtTime(sub.updatedAt) }}
                    </template>
                    <template v-else>未更新节点缓存</template>
                  </div>
                </div>
                <div class="sub-actions" @click.stop>
                  <button class="icon-btn secondary"
                    :class="{ loading: subsStore.updating[sub.id] }"
                    :disabled="!!subsStore.updating[sub.id]"
                    @click="refreshSub(sub.id)" title="重新拉取节点">
                    {{ subsStore.updating[sub.id] ? '…' : '↻' }}
                  </button>
                  <button class="icon-btn secondary" @click="openWizard(sub)" title="编辑配置">✎</button>
                  <button class="icon-btn danger" :disabled="isRunning"
                    @click="deleteSub(sub.id)" title="删除">✕</button>
                </div>
              </div>
              <div v-if="selectedSubId === sub.id" class="sub-expanded">
                <div v-if="!sub.nodeCount" class="sub-no-nodes">
                  <span>节点缓存为空</span>
                  <button class="icon-btn secondary" style="margin-left:8px"
                    :disabled="!!subsStore.updating[sub.id]"
                    @click="refreshSub(sub.id)">
                    {{ subsStore.updating[sub.id] ? '拉取中…' : '立即拉取节点' }}
                  </button>
                </div>
                <template v-else>
                  <div class="sub-proxy-count">
                    ✓ {{ sub.nodeCount }} 个节点已缓存
                  </div>
                  <div class="sub-start-hint">选中此订阅后点击下方「启动」即可</div>
                </template>
              </div>
            </div>
          </div>
        </div>
        <div class="section" v-if="selectedSubId">
          <div class="section-title">代理模式</div>
          <div class="proxy-grid">
            <button v-for="pm in proxyModes" :key="pm.v"
              class="grid-btn" :class="{ on: proxyMode === pm.v }"
              :disabled="isRunning" @click="proxyMode = pm.v">
              <span class="grid-icon">{{ pm.icon }}</span>
              <span class="grid-name">{{ pm.name }}</span>
              <span class="grid-desc">{{ pm.desc }}</span>
            </button>
          </div>
        </div>
      </template>

      <!-- UPLOAD MODE -->
      <template v-if="mode === 'upload'">
        <div class="section">
          <div class="section-title">上传配置文件</div>
          <label class="upload-drop" :class="{ over: dragOver }"
            @dragover.prevent="dragOver = true"
            @dragleave="dragOver = false"
            @drop.prevent="onDrop">
            <input type="file" accept=".json" style="display:none"
              @change="onFileSelect" ref="fileInput" />
            <span class="upload-icon">📁</span>
            <span class="upload-label">拖拽或点击上传 sing-box JSON 配置</span>
          </label>
          <div v-if="uploadErr" class="error-bar" style="margin-top:6px">{{ uploadErr }}</div>
          <div v-if="uploadOk"  class="info-bar"  style="margin-top:6px">{{ uploadOk }}</div>
        </div>

        <div class="section">
          <div class="section-title-row">
            <span class="section-title">已上传配置</span>
            <span class="config-count">{{ uploadedConfigs.length }} 个文件</span>
          </div>
          <div v-if="uploadedConfigs.length === 0" class="empty-tip">暂无上传配置</div>
          <div v-else class="config-list">
            <div v-for="cfg in uploadedConfigs" :key="cfg.filename" class="config-item">
              <div class="config-item-header">
                <span class="config-filename">{{ cfg.filename }}</span>
                <div class="config-actions">
                  <button class="icon-btn secondary" @click="viewConfig(cfg)" title="查看/编辑">✎</button>
                  <button class="icon-btn danger" @click="deleteConfig(cfg.filename)" title="删除">✕</button>
                </div>
              </div>
              <div class="config-meta">
                <span>{{ fmtSize(cfg.size) }}</span>
                <span>{{ fmtTime(cfg.updatedAt) }}</span>
                <span v-if="cfg.inbounds && cfg.inbounds.length">{{ cfg.inbounds.length }} 个入站</span>
              </div>
            </div>
          </div>
        </div>
      </template>

      <div v-if="startError" class="error-bar">{{ startError }}</div>

      <div v-if="isRunning && statusStore.status.ports" class="info-grid">
        <template v-if="statusStore.status.ports.mixed">
          <span class="info-k">HTTP/SOCKS5</span>
          <span class="info-v">127.0.0.1:{{ statusStore.status.ports.mixed }}</span>
        </template>
        <template v-if="statusStore.status.ports.tproxy">
          <span class="info-k">TProxy</span>
          <span class="info-v">:{{ statusStore.status.ports.tproxy }}</span>
        </template>
        <template v-if="statusStore.status.ports.dns">
          <span class="info-k">DNS</span>
          <span class="info-v">:{{ statusStore.status.ports.dns }}</span>
        </template>
      </div>

      <div class="section action-section">
        <button v-if="!isRunning"
          class="btn-start" :disabled="!canStart || starting" @click="doStart">
          {{ starting ? '启动中…' : '▶  启动' }}
        </button>
        <button v-else class="btn-stop" @click="doStop">
          ⏹  停止
        </button>
      </div>

    </div>

    <!-- Import nodes modal -->
    <div v-if="showImport" class="mask" @click.self="showImport = false">
      <div class="modal">
        <div class="modal-head">
          <span>导入节点</span>
          <button class="modal-x" @click="showImport = false">✕</button>
        </div>
        <div class="modal-body">
          <p class="modal-hint">
            粘贴分享链接（每行一个），支持
            vmess:// vless:// trojan:// ss:// tuic:// hy2://
          </p>
          <textarea class="import-ta" v-model="importText"
            placeholder="vmess://...&#10;vless://..."></textarea>
          <div v-if="importErrs.length" class="import-errs">
            <div v-for="(e, i) in importErrs" :key="i" class="import-err">⚠ {{ e }}</div>
          </div>
        </div>
        <div class="modal-foot">
          <button class="btn-cancel" @click="showImport = false">取消</button>
          <button class="btn-ok"
            :disabled="!importText.trim() || importing" @click="doImport">
            {{ importing ? '导入中…' : '导入' }}
          </button>
        </div>
      </div>
    </div>

    <SubWizard v-if="showWizard"
      :sub="editingSub"
      @close="showWizard = false"
      @saved="onWizardSaved" />

    <!-- 查看/编辑配置弹窗 -->
    <div v-if="editingConfig" class="mask" @click.self="closeEditor">
      <div class="modal modal-wide">
        <div class="modal-head">
          <span>{{ editingConfig.filename }}</span>
          <button class="modal-x" @click="closeEditor">✕</button>
        </div>
        <div class="modal-body" style="padding:0;display:flex;flex-direction:column;flex:1;overflow:hidden">
          <textarea class="json-editor" v-model="editingContent" spellcheck="false"></textarea>
          <div v-if="editErr" class="error-bar" style="margin:8px 12px 0">{{ editErr }}</div>
          <div v-if="editOk"  class="info-bar"  style="margin:8px 12px 0">{{ editOk }}</div>
        </div>
        <div class="modal-foot">
          <button class="btn-cancel" @click="closeEditor">关闭</button>
          <button class="btn-ok" :disabled="saving" @click="saveEdit">
            {{ saving ? '保存中…' : '保存' }}
          </button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api.js'
import { useStatusStore, useNodesStore, useSubsStore, useLogsStore } from '../stores.js'
import SubWizard from '../components/SubWizard.vue'

const statusStore = useStatusStore()
const nodesStore  = useNodesStore()
const subsStore   = useSubsStore()
const logsStore   = useLogsStore()

const isRunning = computed(() => statusStore.isRunning)
const mode = ref('node')

const proxyMode = ref('system_proxy')
const routeMode = ref('whitelist')
const lanProxy  = ref(false)
const ipv6      = ref(false)
const blockAds  = ref(true)

const proxyModes = [
  { v: 'system_proxy', icon: '🖥', name: 'system_proxy', desc: '系统代理' },
  { v: 'tproxy',       icon: '🔀', name: 'tproxy',       desc: '透明代理 (Linux)' },
  { v: 'redirect',     icon: '↩', name: 'redirect',      desc: '重定向 (iptables)' },
  { v: 'tun',          icon: '🌐', name: 'tun',           desc: 'TUN 虚拟网卡' },
]
const routeModes = [
  { v: 'whitelist', icon: '🇨🇳', name: '绕过大陆', desc: '国内直连，国外代理' },
  { v: 'gfwlist',   icon: '📋',  name: 'GFW 列表', desc: '仅代理被墙域名' },
  { v: 'global',    icon: '🌍',  name: '全局代理', desc: '所有流量走代理' },
]

// Node mode
const selectedNodeId = ref(null)
const showImport     = ref(false)
const importText     = ref('')
const importErrs     = ref([])
const importing      = ref(false)

async function doImport() {
  if (!importText.value.trim()) return
  importing.value = true
  importErrs.value = []
  try {
    const res = await nodesStore.importNodes(importText.value.trim())
    importErrs.value = res.errors || []
    if ((res.nodes || []).length > 0) {
      importText.value = ''
      showImport.value = false
      if (nodesStore.nodes.length > 0) {
        selectedNodeId.value = nodesStore.nodes[nodesStore.nodes.length - 1].id
      }
    }
  } catch (e) {
    importErrs.value = [e.message]
  } finally {
    importing.value = false
  }
}

async function deleteNode(id) {
  if (selectedNodeId.value === id) selectedNodeId.value = null
  await nodesStore.deleteNode(id)
}

// Subscription mode
const selectedSubId = ref(null)
const showWizard    = ref(false)
const editingSub    = ref(null)

function openWizard(sub) { editingSub.value = sub; showWizard.value = true }

async function onWizardSaved(sub) {
  showWizard.value = false
  await subsStore.load()
  selectedSubId.value = sub.id
  if (!sub.nodeCount) await refreshSub(sub.id)
}

async function refreshSub(id) { try { await subsStore.update(id) } catch {} }
async function deleteSub(id) {
  if (selectedSubId.value === id) selectedSubId.value = null
  await subsStore.remove(id)
}

function fmtTime(iso) {
  if (!iso) return ''
  return new Date(iso).toLocaleString('zh-CN', {
    month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit',
  })
}
function fmtSize(bytes) {
  if (!bytes) return '0 B'
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / 1024 / 1024).toFixed(1) + ' MB'
}

// Upload mode
const uploadedConfigs = ref([])
const uploadErr  = ref('')
const uploadOk   = ref('')
const dragOver   = ref(false)
const fileInput  = ref(null)

function getAuthHeaders() {
  const token = localStorage.getItem('singa_token') || ''
  return token ? { 'X-Auth-Token': token } : {}
}

async function loadUploadedConfigs() {
  try { uploadedConfigs.value = await api('GET', '/config/list') } catch {}
}

async function uploadConfig(file) {
  uploadErr.value = ''; uploadOk.value = ''
  if (!file) return
  if (!file.name.endsWith('.json')) { uploadErr.value = '只允许上传 .json 文件'; return }
  try {
    const fd = new FormData()
    fd.append('config', file)
    const res = await fetch('/api/config', {
      method: 'POST', body: fd,
      headers: getAuthHeaders()
    })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error)
    uploadOk.value = `✓ 上传成功：${data.filename}`
    await loadUploadedConfigs()
  } catch (e) { uploadErr.value = '✕ ' + e.message }
}

function onFileSelect(e) { uploadConfig(e.target.files[0]); e.target.value = '' }
function onDrop(e) { dragOver.value = false; uploadConfig(e.dataTransfer.files[0]) }

async function deleteConfig(filename) {
  if (!confirm('确定删除 ' + filename + '？')) return
  try {
    await api('DELETE', '/config/' + encodeURIComponent(filename))
    await loadUploadedConfigs()
  } catch (e) { alert('删除失败：' + e.message) }
}

// Editor
const editingConfig  = ref(null)
const editingContent = ref('')
const editErr        = ref('')
const editOk         = ref('')
const saving         = ref(false)

async function viewConfig(cfg) {
  editErr.value = ''; editOk.value = ''
  try {
    const res = await fetch('/api/config/raw/' + encodeURIComponent(cfg.filename), {
      headers: getAuthHeaders()
    })
    const text = await res.text()
    editingContent.value = JSON.stringify(JSON.parse(text), null, 2)
  } catch {
    editingContent.value = ''
  }
  editingConfig.value = cfg
}

function closeEditor() {
  editingConfig.value = null
  editingContent.value = ''
  editErr.value = ''
  editOk.value = ''
}

async function saveEdit() {
  editErr.value = ''; editOk.value = ''
  try { JSON.parse(editingContent.value) } catch { editErr.value = 'JSON 格式错误'; return }
  saving.value = true
  try {
    const res = await fetch('/api/config/raw/' + encodeURIComponent(editingConfig.value.filename), {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json', ...getAuthHeaders() },
      body: editingContent.value
    })
    const data = await res.json()
    if (!res.ok) throw new Error(data.error)
    editOk.value = '✓ 已保存'
    await loadUploadedConfigs()
  } catch (e) { editErr.value = '✕ ' + e.message }
  finally { saving.value = false }
}

const selectedSub = computed(() => subsStore.subs.find(s => s.id === selectedSubId.value) ?? null)

const canStart = computed(() => {
  if (mode.value === 'node')         return !!selectedNodeId.value
  if (mode.value === 'subscription') return !!selectedSubId.value && !!selectedSub.value?.nodeCount
  if (mode.value === 'upload')       return uploadedConfigs.value.length > 0
  return false
})

const starting   = ref(false)
const startError = ref('')

async function doStart() {
  if (!canStart.value) return
  starting.value = true; startError.value = ''
  try {
    const base = { proxyMode: proxyMode.value, lanProxy: lanProxy.value, ipv6: ipv6.value, blockAds: blockAds.value, routeMode: routeMode.value }
    if (mode.value === 'node') {
      await api('POST', '/start', { ...base, configMode: 'node', nodeId: selectedNodeId.value })
    } else if (mode.value === 'subscription') {
      await api('POST', '/start', { ...base, configMode: 'subscription', subscriptionId: selectedSubId.value })
    } else {
      await api('POST', '/start', { ...base, configMode: 'upload', uploadedConfigFile: uploadedConfigs.value[0]?.filename || '' })
    }
    await statusStore.fetch()
    logsStore.startSSE()
  } catch (e) { startError.value = e.message }
  finally { starting.value = false }
}

async function doStop() { await statusStore.stop(); logsStore.stopSSE() }

onMounted(async () => {
  await Promise.all([ nodesStore.load(), subsStore.load(), loadUploadedConfigs() ])
  if (nodesStore.nodes.length > 0) selectedNodeId.value = nodesStore.nodes[0].id
  if (subsStore.subs.length > 0) selectedSubId.value = subsStore.subs[0].id
})
</script>

<style scoped>
.mode-grid {
  display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 6px;
}
.mode-btn {
  display: flex; flex-direction: column; align-items: flex-start; gap: 2px;
  padding: 10px 12px; border: 1.5px solid var(--border); border-radius: var(--radius);
  background: var(--bg); cursor: pointer; transition: all .15s;
  text-align: left; color: var(--text2);
}
.mode-btn:hover { border-color: var(--border2); color: var(--text); background: var(--surface); }
.mode-btn.on    { border-color: var(--accent); color: var(--accent); background: var(--accent-bg); }
.mode-icon { font-size: 18px; line-height: 1; margin-bottom: 3px; }
.mode-name { font-size: 12px; font-weight: 700; }
.mode-desc { font-size: 10px; opacity: .7; line-height: 1.3; }
.proxy-grid { grid-template-columns: 1fr 1fr; }
.sub-expanded { border-top: 1px solid var(--border); padding: 10px 12px; background: var(--bg); font-size: 12px; }
.sub-no-nodes { display: flex; align-items: center; gap: 4px; color: var(--text3); }
.sub-proxy-count { color: var(--accent); font-weight: 600; margin-bottom: 4px; }
.sub-start-hint  { color: var(--text3); font-size: 11px; }
.upload-drop {
  display: flex; flex-direction: column; align-items: center; justify-content: center;
  gap: 8px; padding: 24px 16px;
  border: 2px dashed var(--border2); border-radius: var(--radius);
  cursor: pointer; transition: all .15s; margin-top: 6px; background: var(--bg);
}
.upload-drop:hover, .upload-drop.over { border-color: var(--accent); background: var(--accent-bg); }
.upload-icon  { font-size: 28px; }
.upload-label { font-size: 12px; color: var(--text3); text-align: center; }
.config-count { font-size: 11px; color: var(--text3); }
.config-list { display: flex; flex-direction: column; gap: 6px; margin-top: 6px; }
.config-item {
  border: 1.5px solid var(--border); border-radius: var(--radius);
  background: var(--surface); padding: 8px 10px;
}
.config-item-header { display: flex; align-items: center; justify-content: space-between; }
.config-filename {
  font-size: 12px; font-weight: 600; color: var(--text1);
  font-family: var(--mono); word-break: break-all; flex: 1;
}
.config-actions { display: flex; gap: 4px; flex-shrink: 0; margin-left: 8px; }
.config-meta { display: flex; gap: 8px; margin-top: 4px; font-size: 11px; color: var(--text3); flex-wrap: wrap; }
.action-section { flex-direction: row !important; }
.modal-wide {
  width: 90vw; max-width: 820px; max-height: 90vh;
  display: flex; flex-direction: column;
}
.json-editor {
  width: 100%; min-height: 420px; flex: 1; resize: vertical;
  font-family: var(--mono); font-size: 12px; line-height: 1.5;
  padding: 12px; background: #0f1117; color: #e2e8f0;
  border: none; outline: none; box-sizing: border-box;
}
</style>
