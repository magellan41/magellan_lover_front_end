<template>
  <div class="agent-config">
    <div v-if="loading" class="loading-tip">加载配置中...</div>
    <template v-else>
      <div v-if="message" :class="['message-bar', messageType]">{{ message }}</div>

      <div class="config-scroll">
        <!-- 大模型配置 -->
        <div class="config-section">
          <div class="section-header">
            <h3>大模型配置</h3>
            <button class="btn-ghost" @click="addPlatform">+ 新增平台</button>
          </div>

          <div v-for="(p, pi) in platforms" :key="pi" class="platform-card">
            <div class="platform-head">
              <span class="platform-index">#{{ pi + 1 }}</span>
              <button class="btn-ghost-danger" @click="removePlatform(pi)">删除平台</button>
            </div>

            <div class="field-row">
              <label>平台名称</label>
              <input v-model="p.platform" class="field-input" placeholder="如 openai" />
            </div>
            <div class="field-row">
              <label>Base URL</label>
              <input v-model="p.base_url" class="field-input" placeholder="https://api..." />
            </div>
            <div class="field-row">
              <label>Key 类型</label>
              <select v-model="p.key_type" class="field-input">
                <option value="env">env（环境变量）</option>
                <option value="str">str（明文）</option>
              </select>
            </div>
            <div class="field-row">
              <label>API Key</label>
              <input v-model="p.api_key" class="field-input" placeholder="API Key 或环境变量名" />
            </div>

            <!-- Models -->
            <div class="models-section">
              <div class="models-header">
                <span>模型列表</span>
                <button class="btn-ghost" @click="addModel(pi)">+ 新增模型</button>
              </div>
              <div v-for="(m, mi) in p.models" :key="mi" class="model-card">
                <div class="model-head">
                  <span>{{ m.name || '新模型' }}</span>
                  <button class="btn-ghost-danger" @click="removeModel(pi, mi)">删除</button>
                </div>
                <div class="field-row">
                  <label>模型名称</label>
                  <input v-model="m.name" class="field-input" placeholder="model name" />
                </div>
                <div class="field-row">
                  <label>输入类型</label>
                  <div class="checkbox-group">
                    <label v-for="t in inputTypeOptions" :key="t" class="checkbox-label">
                      <input type="checkbox" :value="t" v-model="m.input_type" />
                      {{ t }}
                    </label>
                  </div>
                </div>
                <div class="field-row">
                  <label>最大上下文</label>
                  <input v-model.number="m.max_context_windows" type="number" class="field-input" placeholder="如 128000" />
                </div>
              </div>
            </div>
          </div>

          <div class="section-save">
            <button class="btn-ghost-primary" :disabled="savingLlm" @click="saveLlmConfig">
              {{ savingLlm ? '保存中...' : '保存大模型配置' }}
            </button>
          </div>
        </div>

        <!-- Agent 配置 -->
        <div class="config-section">
          <div class="section-header">
            <h3>Agent 配置</h3>
          </div>
          <div class="field-row">
            <label>对话模型 (chat)</label>
            <select v-model="agents.chat" class="field-input">
              <option v-for="opt in modelOptions" :key="opt" :value="opt">{{ opt }}</option>
            </select>
          </div>
          <div class="field-row">
            <label>压缩模型 (compact)</label>
            <select v-model="agents.compact" class="field-input">
              <option v-for="opt in modelOptions" :key="opt" :value="opt">{{ opt }}</option>
            </select>
          </div>
          <div class="field-row">
            <label>记忆模型 (memory)</label>
            <select v-model="agents.memory" class="field-input">
              <option v-for="opt in modelOptions" :key="opt" :value="opt">{{ opt }}</option>
            </select>
          </div>
          <div class="field-row">
            <label>日程模型 (story)</label>
            <select v-model="agents.story" class="field-input">
              <option v-for="opt in modelOptions" :key="opt" :value="opt">{{ opt }}</option>
            </select>
          </div>
          <div class="section-save">
            <button class="btn-ghost-primary" :disabled="savingAgent" @click="saveAgentConfig">
              {{ savingAgent ? '保存中...' : '保存 Agent 配置' }}
            </button>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
export default {
  name: 'ConfigAgent',
  data() {
    return {
      loading: false,
      message: '',
      messageType: 'success',
      savingLlm: false,
      savingAgent: false,
      platforms: [],
      agents: { chat: '', compact: '', memory: '', story: '' },
      inputTypeOptions: ['text', 'image', 'audio', 'video']
    }
  },
  computed: {
    modelOptions() {
      const opts = []
      for (const p of this.platforms) {
        if (!p.platform) continue
        for (const m of (p.models || [])) {
          if (m.name) opts.push(`${p.platform}/${m.name}`)
        }
      }
      return opts
    }
  },
  mounted() {
    this.loadConfig()
  },
  methods: {
    async loadConfig() {
      this.loading = true
      try {
        const res = await fetch('/api/agent/config/get', { cache: 'no-cache' })
        const text = await res.text()
        let data
        try {
          data = JSON.parse(text)
          if (typeof data === 'string') data = JSON.parse(data)
        } catch (e) {
          data = JSON.parse(JSON.parse(text))
        }
        this.platforms = (data.platforms || []).map(p => ({
          ...p,
          models: (p.models || []).map(m => ({
            ...m,
            input_type: Array.isArray(m.input_type) ? [...m.input_type] : []
          }))
        }))
        this.agents = { chat: '', compact: '', memory: '', story: '', ...(data.agents || {}) }
      } catch (e) {
        console.error('加载 Agent 配置失败:', e)
        this.showMessage('加载配置失败: ' + e.message, 'error')
      } finally {
        this.loading = false
      }
    },
    addPlatform() {
      this.platforms.push({
        platform: '',
        base_url: '',
        key_type: 'env',
        api_key: '',
        models: []
      })
    },
    removePlatform(index) {
      if (confirm('确定删除此平台？')) {
        this.platforms.splice(index, 1)
      }
    },
    addModel(platformIndex) {
      this.platforms[platformIndex].models.push({
        name: '',
        input_type: ['text'],
        max_context_windows: 128000
      })
    },
    removeModel(platformIndex, modelIndex) {
      this.platforms[platformIndex].models.splice(modelIndex, 1)
    },
    buildPayload() {
      return {
        platforms: this.platforms.map(p => ({
          platform: p.platform,
          base_url: p.base_url,
          key_type: p.key_type,
          api_key: p.api_key,
          models: p.models.map(m => ({
            name: m.name,
            input_type: m.input_type,
            max_context_windows: Number(m.max_context_windows) || 0
          }))
        })),
        agents: { ...this.agents }
      }
    },
    async saveLlmConfig() {
      this.savingLlm = true
      try {
        const payload = this.buildPayload()
        const res = await fetch('/api/agent/config/set', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(JSON.stringify(payload))
        })
        if (res.ok) {
          this.showMessage('大模型配置保存成功！', 'success')
        } else {
          this.showMessage('保存失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('保存失败: 网络错误', 'error')
      } finally {
        this.savingLlm = false
      }
    },
    async saveAgentConfig() {
      this.savingAgent = true
      try {
        const payload = this.buildPayload()
        const res = await fetch('/api/agent/config/set', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(JSON.stringify(payload))
        })
        if (res.ok) {
          this.showMessage('Agent 配置保存成功！', 'success')
        } else {
          this.showMessage('保存失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('保存失败: 网络错误', 'error')
      } finally {
        this.savingAgent = false
      }
    },
    showMessage(text, type) {
      this.message = text
      this.messageType = type
      setTimeout(() => { this.message = '' }, 3000)
    }
  }
}
</script>

<style scoped>
.agent-config {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: #f7f8fa;
}
.loading-tip { text-align: center; color: #b0b0b0; padding: 40px; }

.message-bar { padding: 8px 16px; font-size: 13px; text-align: center; flex-shrink: 0; }
.message-bar.success { background: #f2f9f2; color: #4a9a5b; }
.message-bar.error { background: #fdf2f2; color: #c75050; }

.config-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
}

.config-section {
  background: #fff;
  border-radius: 10px;
  margin-bottom: 16px;
  border: 1px solid #eef0f2;
  overflow: hidden;
  padding-bottom: 12px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.03);
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #fafbfc;
  border-bottom: 1px solid #eef0f2;
}
.section-header h3 { margin: 0; font-size: 14px; font-weight: 600; color: #444; }

/* 圆角虚线按钮 */
.btn-ghost {
  padding: 4px 14px;
  border: 1px dashed #b8d4ec;
  border-radius: 100px;
  font-size: 12px;
  cursor: pointer;
  background: transparent;
  color: #6aabd8;
  transition: all 0.25s;
}
.btn-ghost:hover {
  border-color: #5b9bd5;
  color: #4a90c4;
  background: rgba(91, 155, 213, 0.05);
}
.btn-ghost-primary {
  padding: 5px 18px;
  border: 1px dashed #b8d4ec;
  border-radius: 100px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  background: transparent;
  color: #6aabd8;
  transition: all 0.25s;
}
.btn-ghost-primary:hover {
  background: rgba(91, 155, 213, 0.06);
  border-color: #5b9bd5;
}
.btn-ghost-danger {
  padding: 4px 14px;
  border: 1px dashed #f0b3be;
  border-radius: 100px;
  font-size: 12px;
  cursor: pointer;
  background: transparent;
  color: #d4768a;
  transition: all 0.25s;
}
.btn-ghost-danger:hover {
  background: rgba(212, 118, 138, 0.06);
  border-color: #d4768a;
}
.btn-ghost:disabled,
.btn-ghost-primary:disabled,
.btn-ghost-danger:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
.btn-ghost-primary:disabled:hover,
.btn-ghost-danger:disabled:hover {
  background: transparent;
}

/* Platform card */
.platform-card {
  margin: 12px 16px;
  border: 1px solid #eef0f2;
  border-radius: 10px;
  padding: 14px;
  background: #fafbfc;
}
.platform-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}
.platform-index {
  font-size: 13px;
  font-weight: 600;
  color: #6aabd8;
}

/* Field rows */
.field-row {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 8px;
}
.field-row label {
  min-width: 80px;
  font-size: 13px;
  font-weight: 500;
  color: #666;
  flex-shrink: 0;
}
.field-input {
  flex: 1;
  padding: 7px 10px;
  border: 1px solid #e4e8ec;
  border-radius: 8px;
  font-size: 13px;
  outline: none;
  transition: border-color 0.25s, box-shadow 0.25s;
  background: #fff;
  color: #444;
}
.field-input:focus {
  border-color: #b8cfe8;
  box-shadow: 0 0 0 3px rgba(91, 155, 213, 0.08);
}

/* Checkbox group */
.checkbox-group {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}
.checkbox-label {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 13px;
  color: #666;
  cursor: pointer;
}
.checkbox-label input[type="checkbox"] {
  accent-color: #8bbde6;
}

/* Models section */
.models-section {
  margin-top: 10px;
  border-top: 1px solid #eef0f2;
  padding-top: 10px;
}
.models-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}
.models-header span { font-size: 13px; font-weight: 600; color: #666; }

.model-card {
  margin: 8px 0;
  border: 1px solid #eef0f2;
  border-radius: 8px;
  padding: 10px;
  background: #fff;
}
.model-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-size: 13px;
  font-weight: 500;
  color: #666;
}

.section-save {
  padding: 12px 16px 0;
  text-align: center;
}
</style>
