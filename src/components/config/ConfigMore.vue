<template>
  <div class="config-more">
    <div v-if="loading" class="loading-tip">加载配置中...</div>
    <template v-else>
      <div v-if="message" :class="['message-bar', messageType]">{{ message }}</div>

      <div class="config-scroll">
        <!-- 日程描述 -->
        <div class="config-section">
          <div class="section-header">
            <h3>日程描述</h3>
          </div>
          <div class="field-col">
            <textarea
              v-model="scheduleDescription"
              class="field-textarea"
              rows="6"
              placeholder="请输入 Agent 的日程描述..."
            ></textarea>
          </div>
          <div class="section-save">
            <button class="btn-ghost-primary" :disabled="savingSchedule" @click="saveSchedule">
              {{ savingSchedule ? '保存中...' : '保存日程描述' }}
            </button>
          </div>
        </div>

        <!-- 图片生成工具 -->
        <div class="config-section">
          <div class="section-header">
            <h3>图片生成工具</h3>
          </div>

          <div class="field-row">
            <label>平台</label>
            <select v-model="imageGeneratorPlatform" class="field-input">
              <option value="" disabled>请选择平台</option>
              <option v-for="opt in platformOptions" :key="opt" :value="opt">{{ opt }}</option>
            </select>
          </div>

          <div class="field-row">
            <label>API Key</label>
            <input
              v-model="imageGeneratorApiKey"
              class="field-input"
              placeholder="API Key 或环境变量名"
            />
          </div>

          <div class="field-row">
            <label>模型名称</label>
            <input
              v-model="imageGeneratorModel"
              class="field-input"
              placeholder="如 dall-e-3"
            />
          </div>

          <div class="field-row">
            <label>基准图片</label>
            <div class="char-image-area">
              <img v-if="characterImageSrc" :src="characterImageSrc" class="char-image-preview" @click="previewCharImage" />
              <div v-else class="char-image-empty">未上传</div>
              <label class="upload-char-btn">
                <input type="file" accept="image/png,image/jpeg,image/jpg" class="upload-char-input" @change="uploadCharacterImage" />
                {{ characterImageSrc ? '重新上传' : '上传图片' }}
              </label>
            </div>
          </div>

          <div class="section-save">
            <button class="btn-ghost-primary" :disabled="savingImage" @click="saveImageConfig">
              {{ savingImage ? '保存中...' : '保存图片生成配置' }}
            </button>
          </div>
        </div>

        <!-- 语音设置 -->
        <div class="config-section">
          <div class="section-header">
            <h3>语音设置</h3>
          </div>

          <div class="field-row">
            <label>启用语音</label>
            <label class="switch">
              <input type="checkbox" v-model="voiceEnable" />
              <span class="slider"></span>
            </label>
          </div>

          <div class="field-row" :class="{ disabled: !voiceEnable }">
            <label>API Key</label>
            <input
              v-model="voiceApiKey"
              class="field-input"
              :disabled="!voiceEnable"
              placeholder="API Key 或环境变量名"
            />
          </div>

          <div class="field-row" :class="{ disabled: !voiceEnable }">
            <label>语音生成器</label>
            <select v-model="voiceGenerator" class="field-input" :disabled="!voiceEnable">
              <option value="minimax">minimax</option>
            </select>
          </div>

          <div class="section-save">
            <button class="btn-ghost-primary" :disabled="saving" @click="saveConfig">
              {{ saving ? '保存中...' : '保存语音设置' }}
            </button>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { BACKEND_BASE } from '@/config'

export default {
  name: 'ConfigMore',
  data() {
    return {
      loading: false,
      saving: false,
      message: '',
      messageType: 'success',
      voiceEnable: false,
      voiceApiKey: '',
      voiceGenerator: 'minimax',
      scheduleDescription: '',
      savingSchedule: false,
      imageGeneratorPlatform: '',
      imageGeneratorApiKey: '',
      imageGeneratorModel: '',
      platformOptions: [],
      characterImageSrc: '',
      savingImage: false,
      original: '',
      ready: false
    }
  },
  computed: {
    hasChanges() {
      return this.buildPayload() !== this.original
    }
  },
  mounted() {
    this.loadConfig()
    this.loadSchedule()
    this.loadImageConfig()
    this.loadPlatformOptions()
  },
  watch: {
    voiceEnable() {
      // 加载完成后才响应，且值真正发生变化才保存
      if (this.ready && this.hasChanges) this.saveConfig()
    }
  },
  methods: {
    async loadConfig() {
      this.loading = true
      try {
        const res = await fetch('/api/agent/voice/get', { cache: 'no-cache' })
        if (res.ok) {
          const data = await res.json()
          // 先标记 ready，再赋值；watcher 触发时 ready 仍为 false，跳过首次保存
          this.ready = true
          this.voiceEnable = this.parseBool(data.voice_enable)
          this.voiceApiKey = data.voice_api_key || ''
          this.voiceGenerator = data.voice_generator || 'minimax'
          this.original = this.buildPayload()
        }
      } catch (e) {
        this.showMessage('加载配置失败', 'error')
      } finally {
        this.loading = false
      }
    },
    parseBool(val) {
      if (typeof val === 'boolean') return val
      const s = String(val).toLowerCase().replace(/["']/g, '')
      return s === 'true'
    },
    buildPayload() {
      return JSON.stringify({
        voice_enable: String(this.voiceEnable),
        voice_api_key: this.voiceApiKey,
        voice_generator: this.voiceGenerator
      })
    },
    async saveConfig() {
      if (this.saving) return
      this.saving = true
      try {
        const res = await fetch('/api/agent/voice/set', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            voice_enable: String(this.voiceEnable),
            voice_api_key: this.voiceApiKey,
            voice_generation_type: this.voiceGenerator
          })
        })
        if (res.ok) {
          this.original = this.buildPayload()
          this.showMessage('保存成功！', 'success')
        } else {
          this.showMessage('保存失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('保存失败: 网络错误', 'error')
      } finally {
        this.saving = false
      }
    },
    async loadSchedule() {
      try {
        const res = await fetch('/api/agent/schedule_description/get', { cache: 'no-cache' })
        if (res.ok) {
          // 后端 -> str 返回时 FastAPI 自动 JSON 编码（"text"），需 parse 去引号
          const text = await res.text()
          try { this.scheduleDescription = JSON.parse(text) } catch (e) { this.scheduleDescription = text }
        }
      } catch (e) {
        this.showMessage('加载日程描述失败', 'error')
      }
    },
    async saveSchedule() {
      if (this.savingSchedule) return
      this.savingSchedule = true
      try {
        // 后端 Body(..., description=...) 期望 JSON 编码的字符串
        const res = await fetch('/api/agent/schedule_description/set', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(this.scheduleDescription)
        })
        if (res.ok) {
          this.showMessage('日程描述保存成功！', 'success')
        } else {
          this.showMessage('保存失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('保存失败: 网络错误', 'error')
      } finally {
        this.savingSchedule = false
      }
    },
    showMessage(text, type) {
      this.message = text
      this.messageType = type
      setTimeout(() => { this.message = '' }, 3000)
    },
    async fetchEnv(key) {
      const res = await fetch(`/api/agent/env/get/${key}`, { cache: 'no-cache' })
      if (!res.ok) return ''
      const text = await res.text()
      try { return JSON.parse(text) } catch (e) { return text }
    },
    async setEnv(key, value) {
      const res = await fetch('/api/agent/env/set', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ key, value })
      })
      return res.ok
    },
    async loadImageConfig() {
      try {
        const [platform, apiKey, model] = await Promise.all([
          this.fetchEnv('image_generator_platform'),
          this.fetchEnv('image_generator_api_key'),
          this.fetchEnv('image_generator_model')
        ])
        this.imageGeneratorPlatform = platform
        this.imageGeneratorApiKey = apiKey
        this.imageGeneratorModel = model
      } catch (e) {
        this.showMessage('加载图片生成配置失败', 'error')
      }
      this.probeCharacterImage()
    },
    async saveImageConfig() {
      if (this.savingImage) return
      this.savingImage = true
      try {
        const [okPlatform, okKey, okModel] = await Promise.all([
          this.setEnv('image_generator_platform', this.imageGeneratorPlatform),
          this.setEnv('image_generator_api_key', this.imageGeneratorApiKey),
          this.setEnv('image_generator_model', this.imageGeneratorModel)
        ])
        if (okPlatform && okKey && okModel) {
          this.showMessage('图片生成配置保存成功！', 'success')
        } else {
          this.showMessage('保存失败', 'error')
        }
      } catch (e) {
        this.showMessage('保存失败: 网络错误', 'error')
      } finally {
        this.savingImage = false
      }
    },
    probeCharacterImage() {
      this.fetchEnv('character_image_url').then(path => {
        if (path) {
          this.characterImageSrc = path.startsWith('/') ? BACKEND_BASE + path : path
        } else {
          this.characterImageSrc = ''
        }
      })
    },
    async loadPlatformOptions() {
      try {
        const res = await fetch('/api/agent/image_generator/platform/get', { cache: 'no-cache' })
        if (res.ok) {
          const text = await res.text()
          let data = text
          try {
            data = JSON.parse(text)
            if (typeof data === 'string') {
              try { data = JSON.parse(data) } catch (e) { /* single layer */ }
            }
          } catch (e) { /* plain text */ }
          if (Array.isArray(data)) {
            this.platformOptions = data
          }
        }
      } catch (e) {
        console.error('加载平台列表失败:', e)
      }
    },
    async uploadCharacterImage(e) {
      const file = e.target.files[0]
      if (!file) return
      e.target.value = ''
      try {
        const formData = new FormData()
        formData.append('file', file)
        const res = await fetch('/api/file/uploads/character_image', { method: 'POST', body: formData })
        if (res.ok) {
          this.probeCharacterImage()
          this.showMessage('基准图片上传成功！', 'success')
        } else {
          this.showMessage('上传失败: ' + res.status, 'error')
        }
      } catch (err) {
        this.showMessage('上传失败: 网络错误', 'error')
      }
    },
    previewCharImage() {
      if (this.characterImageSrc) window.open(this.characterImageSrc, '_blank')
    }
  }
}
</script>

<style scoped>
.config-more {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
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

.field-row {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 16px;
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
.field-input:disabled {
  background: #f5f6f8;
  color: #bbb;
  cursor: not-allowed;
}

.field-col {
  padding: 10px 16px;
}
.field-textarea {
  width: 100%;
  min-height: 120px;
  padding: 10px 12px;
  border: 1px solid #e4e8ec;
  border-radius: 8px;
  font-size: 13px;
  line-height: 1.6;
  outline: none;
  resize: vertical;
  background: #fff;
  color: #444;
  font-family: inherit;
  transition: border-color 0.25s, box-shadow 0.25s;
}
.field-textarea:focus {
  border-color: #b8cfe8;
  box-shadow: 0 0 0 3px rgba(91, 155, 213, 0.08);
}
.field-textarea::placeholder {
  color: #bbb;
}

.field-row.disabled label {
  color: #bbb;
}

/* Toggle switch */
.switch {
  position: relative;
  display: inline-block;
  width: 42px;
  height: 22px;
  flex-shrink: 0;
}
.switch input { opacity: 0; width: 0; height: 0; }
.slider {
  position: absolute;
  cursor: pointer;
  inset: 0;
  background: #d0d5db;
  border-radius: 22px;
  transition: 0.3s;
}
.slider::before {
  content: '';
  position: absolute;
  height: 16px;
  width: 16px;
  left: 3px;
  bottom: 3px;
  background: #fff;
  border-radius: 50%;
  transition: 0.3s;
}
.switch input:checked + .slider { background: #6aabd8; }
.switch input:checked + .slider::before { transform: translateX(16px); }

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
.btn-ghost-primary:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
.btn-ghost-primary:disabled:hover {
  background: transparent;
}

.section-save {
  padding: 12px 16px 0;
  text-align: center;
}

/* Character image */
.char-image-area {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 12px;
}
.char-image-preview {
  width: 64px;
  height: 64px;
  border-radius: 8px;
  object-fit: cover;
  border: 1px solid #e4e8ec;
  cursor: pointer;
  transition: opacity 0.2s;
}
.char-image-preview:hover { opacity: 0.8; }
.char-image-empty {
  width: 64px;
  height: 64px;
  border-radius: 8px;
  border: 1px dashed #d0d5db;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  color: #bbb;
  background: #f9fafb;
  flex-shrink: 0;
}
.upload-char-btn {
  padding: 4px 14px;
  border: 1px dashed #b8d4ec;
  border-radius: 100px;
  font-size: 12px;
  cursor: pointer;
  background: transparent;
  color: #6aabd8;
  transition: all 0.25s;
  flex-shrink: 0;
}
.upload-char-btn:hover {
  border-color: #5b9bd5;
  color: #4a90c4;
  background: rgba(91, 155, 213, 0.05);
}
.upload-char-input {
  position: absolute;
  width: 0;
  height: 0;
  opacity: 0;
  pointer-events: none;
}
</style>
