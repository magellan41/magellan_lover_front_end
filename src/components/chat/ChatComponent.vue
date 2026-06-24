<template>
  <div class="chat-container">
    <div class="chat-header">
      <h2 v-if="!editingName" class="agent-name" @click="startEditName">{{ agentName }}</h2>
      <input
        v-else
        ref="nameInput"
        v-model="agentName"
        class="name-input"
        maxlength="20"
        @blur="saveName"
        @keydown.enter.prevent="saveName"
        @keydown.esc="cancelEditName"
      />
    </div>

    <div class="chat-messages" ref="messagesContainer" @scroll="onScroll">
      <div v-if="loadingMore" class="loading-tip">加载更早的记录...</div>
      <div v-if="loading && messages.length === 0" class="loading-tip">
        加载聊天记录中...
      </div>
      <div v-if="messages.length === 0 && !loading" class="empty-tip">
        暂无聊天记录，发送一条消息开始对话吧
      </div>
      <div
        v-for="(msg, index) in messages"
        :key="msg.id || index"
        :class="['message-row', msg.role === 'user' ? 'message-right' : 'message-left']"
      >
        <label class="avatar" @click.stop>
          <input
            v-if="msg.role === 'user'"
            type="file"
            accept="image/*"
            class="avatar-file-input"
            @change="uploadAvatar($event, 'user')"
          />
          <input
            v-else
            type="file"
            accept="image/*"
            class="avatar-file-input"
            @change="uploadAvatar($event, 'agent')"
          />
          <img v-if="msg.role === 'user' && avatarUser" :src="avatarUser" alt="user" class="avatar-img" />
          <img v-else-if="avatarAgent" :src="avatarAgent" alt="agent" class="avatar-img" />
          <span v-else>{{ msg.role === 'user' ? '我' : 'AI' }}</span>
        </label>
        <div :class="['message-bubble', msg.role === 'user' ? 'bubble-user' : msg.type === 'error' ? 'bubble-error' : 'bubble-agent']">
          <div v-if="msg.type === 'voice' && msg.content" class="voice-player">
            <button class="vp-btn" @click="togglePlay($event, msg)">
              {{ playingId === msg.id && !paused ? '⏸' : '▶' }}
            </button>
            <div class="vp-track" @click="seek($event, msg)">
              <div class="vp-progress" :style="{ width: getProgress(msg) + '%' }"></div>
            </div>
            <span class="vp-time">{{ formatTime(getCurrentTime(msg)) }} / {{ formatTime(durations[msg.id] || 0) }}</span>
            <audio
              :src="getVoiceUrl(msg.content)"
              preload="metadata"
              class="vp-audio"
              @loadedmetadata="onMeta($event, msg)"
              @timeupdate="onTime($event, msg)"
              @ended="onEnded(msg)"
            ></audio>
          </div>
          <div v-else-if="msg.type === 'image' && msg.images && msg.images.length" class="image-grid">
            <img
              v-for="(src, i) in msg.images"
              :key="i"
              :src="src"
              class="chat-image"
              @click="previewImage(src)"
            />
          </div>
          <div v-else-if="msg.type === 'image' && msg.content" class="image-grid">
            <img
              :src="getImageUrl(msg.content)"
              class="chat-image"
              @click="previewImage(getImageUrl(msg.content))"
            />
          </div>
          <pre v-else class="message-text">{{ msg.content }}</pre>
        </div>
      </div>
      <div v-if="sending" class="message-row message-left">
        <div class="avatar">
          <img v-if="avatarAgent" :src="avatarAgent" alt="agent" class="avatar-img" />
          <span v-else>AI</span>
        </div>
        <div class="message-bubble bubble-agent typing">
          <span class="dot"></span>
          <span class="dot"></span>
          <span class="dot"></span>
        </div>
      </div>
    </div>

    <div class="chat-input-area">
      <label class="upload-btn" title="上传图片">
        <input
          type="file"
          accept="image/png,image/jpeg,image/jpg,image/gif"
          multiple
          class="upload-file-input"
          @change="uploadImages"
        />
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <rect x="3" y="3" width="18" height="18" rx="2" ry="2"/>
          <circle cx="8.5" cy="8.5" r="1.5"/>
          <polyline points="21 15 16 10 5 21"/>
        </svg>
      </label>
      <textarea
        v-model="inputMessage"
        class="chat-input"
        placeholder="输入消息..."
        rows="1"
        @keydown.enter.exact.prevent="sendMessage"
        @input="autoResize"
        ref="inputEl"
      ></textarea>
      <button class="send-btn" :disabled="!inputMessage.trim()" @click="sendMessage">
        发送
      </button>
    </div>

    <!-- 图片预览遮罩层 -->
    <div v-if="previewSrc" class="image-overlay" @click="previewSrc = ''">
      <img :src="previewSrc" class="overlay-image" @click.stop />
    </div>
  </div>
</template>

<script>
import { BACKEND_BASE } from '@/config'

export default {
  name: 'ChatComponent',
  data() {
    return {
      messages: [],
      inputMessage: '',
      loading: false,
      loadingMore: false,
      sending: false,
      sseController: null,
      avatarUser: '',
      avatarAgent: '',
      agentName: 'Magellan Chat',
      editingName: false,
      originalName: '',
      minId: -1,
      reachedEnd: false,
      playingId: null,
      paused: false,
      durations: {},
      currents: {},
      previewSrc: ''
    }
  },
  mounted() {
    this.audioRefs = {} // 非响应式，存 audio DOM 元素
    this.fetchAgentName()
    this.fetchAvatars()
    this.fetchHistory()
    this.connectSSE()
  },
  beforeUnmount() {
    this.closeSSE()
  },
  methods: {
    async fetchAgentName() {
      try {
        const res = await fetch('/api/agent/env/get/agent_name', { cache: 'no-cache' })
        if (res.ok) {
          const t = await res.text()
          const val = t.startsWith('"') ? JSON.parse(t) : t
          if (val) {
            this.agentName = val
            document.title = val
          }
        }
      } catch (e) { /* 保持默认名 */ }
    },
    startEditName() {
      this.originalName = this.agentName
      this.editingName = true
      this.$nextTick(() => {
        const inp = this.$refs.nameInput
        if (inp) { inp.focus(); inp.select() }
      })
    },
    async saveName() {
      this.editingName = false
      const name = this.agentName.trim()
      if (!name || name === this.originalName) {
        this.agentName = this.originalName
        return
      }
      try {
        await fetch('/api/agent/env/set', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ key: 'agent_name', value: name })
        })
      } catch (e) {
        this.agentName = this.originalName
      }
    },
    cancelEditName() {
      this.agentName = this.originalName
      this.editingName = false
    },
    async fetchAvatars() {
      try {
        const [resU, resA] = await Promise.all([
          fetch('/api/agent/env/get/avatar_user', { cache: 'no-cache' }),
          fetch('/api/agent/env/get/avatar_agent', { cache: 'no-cache' })
        ])
        if (resU.ok) {
          const t = await resU.text()
          const val = t.startsWith('"') ? JSON.parse(t) : t
          this.avatarUser = val.startsWith('/') ? BACKEND_BASE + val : val
        }
        if (resA.ok) {
          const t = await resA.text()
          const val = t.startsWith('"') ? JSON.parse(t) : t
          this.avatarAgent = val.startsWith('/') ? BACKEND_BASE + val : val
        }
      } catch (e) { /* 头像加载失败 */ }
    },
    async uploadAvatar(event, type) {
      const file = event.target.files[0]
      if (!file) return
      const url = type === 'user'
        ? '/api/file/uploads/avatar/user'
        : '/api/file/uploads/avatar/agent'
      try {
        const formData = new FormData()
        formData.append('file', file)
        const res = await fetch(url, { method: 'POST', body: formData })
        if (res.ok) await this.fetchAvatars()
      } catch (e) { /* 上传失败 */ }
      event.target.value = ''
    },
    async fetchHistory() {
      if (this.loadingMore || this.reachedEnd) return
      const isFirst = this.minId === -1
      this.loading = isFirst
      this.loadingMore = !isFirst
      try {
        const res = await fetch(`/api/chat/list/${this.minId}`, { cache: 'no-cache' })
        const json = await res.json()
        // 兼容多种响应格式：json.data / json.list / json 本身为数组
        const items = Array.isArray(json.data) ? json.data
          : Array.isArray(json.list) ? json.list
          : Array.isArray(json) ? json
          : []
        if (items.length === 0) {
          this.reachedEnd = true
          return
        }
        const newMessages = items.map((item, i) => {
          const type = item.type || 'text'
          const content = item.content || ''
          let images = []
          if (type === 'image' && content) {
            const paths = content.includes(',') ? content.split(',') : [content]
            images = paths.map(p => {
              const trim = p.trim()
              return trim.startsWith('/') ? BACKEND_BASE + trim : trim
            })
          }
          return {
            id: item.id != null ? item.id : `hist_${this.minId}_${i}`,
            role: item.role,
            content,
            type,
            images
          }
        })
        // 加载更多时记录当前滚动高度，用于插入新消息后恢复位置
        const el = this.$refs.messagesContainer
        const prevScrollHeight = el ? el.scrollHeight : 0
        // 倒序后追加到前面（后端返回从近到远，需要反转为从远到近）
        this.messages = [...newMessages.reverse(), ...this.messages]
        // 更新 minId：取所有返回项中的最小 id，兼容数字和字符串类型
        const ids = items.map(i => i.id).filter(id => id != null && !isNaN(Number(id))).map(id => Number(id))
        if (ids.length > 0) {
          this.minId = Math.min(...ids)
        } else {
          this.reachedEnd = true
        }
        if (isFirst) {
          // 首次加载：滚动到底部
          this.scrollToBottom()
        } else {
          // 加载更多：保持当前滚动位置（新消息插入顶部后不跳动）
          this.$nextTick(() => {
            if (el) {
              el.scrollTop = el.scrollHeight - prevScrollHeight
            }
          })
        }
      } catch (e) {
        console.error('获取聊天记录失败:', e)
      } finally {
        this.loading = false
        this.loadingMore = false
      }
    },
    onScroll() {
      const el = this.$refs.messagesContainer
      if (!el || this.loadingMore || this.reachedEnd) return
      if (el.scrollTop < 80) {
        this.fetchHistory()
      }
    },
    getVoiceUrl(content) {
      if (!content) return ''
      return content.startsWith('/') ? BACKEND_BASE + content : content
    },
    getImageUrl(content) {
      if (!content) return ''
      const src = content.trim()
      return src.startsWith('/') ? BACKEND_BASE + src : src
    },
    togglePlay(e, msg) {
      e.stopPropagation()
      const audio = this.audioRefs[msg.id]
      if (!audio) return
      if (this.playingId === msg.id && !this.paused) {
        audio.pause()
        this.paused = true
      } else {
        Object.values(this.audioRefs).forEach(a => a.pause())
        audio.play()
        this.playingId = msg.id
        this.paused = false
      }
    },
    seek(e, msg) {
      const audio = this.audioRefs[msg.id]
      if (!audio || !audio.duration) return
      const rect = e.currentTarget.getBoundingClientRect()
      const ratio = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width))
      audio.currentTime = ratio * audio.duration
    },
    onMeta(e, msg) {
      this.durations = { ...this.durations, [msg.id]: e.target.duration }
      this.audioRefs[msg.id] = e.target
    },
    onTime(e, msg) {
      this.currents = { ...this.currents, [msg.id]: e.target.currentTime }
    },
    onEnded(msg) {
      if (this.playingId === msg.id) {
        this.playingId = null
        this.paused = false
      }
    },
    getProgress(msg) {
      const dur = this.durations[msg.id] || 0
      const cur = this.currents[msg.id] || 0
      return dur > 0 ? (cur / dur) * 100 : 0
    },
    getCurrentTime(msg) {
      return this.currents[msg.id] || 0
    },
    formatTime(sec) {
      const m = Math.floor(sec / 60)
      const s = Math.floor(sec % 60)
      return m + ':' + (s < 10 ? '0' : '') + s
    },
    async sendMessage() {
      const text = this.inputMessage.trim()
      if (!text) return
      this.messages.push({ role: 'user', content: text, type: 'text', id: Date.now() })
      this.inputMessage = ''
      this.resetInputHeight()
      this.scrollToBottom()
      this.$nextTick(() => {
        const el = this.$refs.inputEl
        if (el) el.focus()
      })
      try {
        await fetch('/api/chat/send', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ message: text })
        })
        this.sending = true
      } catch (e) {
        console.error('发送消息失败:', e)
        this.messages.push({ role: 'agent', content: '网络错误，请稍后重试。', type: 'text' })
      }
    },
    async uploadImages(e) {
      const files = Array.from(e.target.files)
      if (!files.length) return
      e.target.value = ''
      const formData = new FormData()
      files.forEach(f => formData.append('files', f))
      try {
        const res = await fetch(BACKEND_BASE + '/api/chat/image', { method: 'POST', body: formData })
        if (res.ok) {
          const data = await res.json()
          const urls = data.urls || []
          const images = urls.map(p => p.startsWith('/') ? BACKEND_BASE + p : p)
          this.messages.push({
            role: 'user',
            type: 'image',
            images,
            content: '',
            id: Date.now()
          })
          this.scrollToBottom()
        } else {
          const err = await res.json().catch(() => ({}))
          const detail = err.detail || '图片上传失败: ' + res.status
          this.messages.push({
            role: 'agent',
            content: '【ERROR】' + detail,
            type: 'error',
            id: Date.now()
          })
          this.scrollToBottom()
        }
      } catch (err) {
        console.error('图片上传失败:', err)
      }
    },
    previewImage(src) {
      this.previewSrc = src
    },
    connectSSE() {
      this.closeSSE()
      this.sseController = new AbortController()
      const signal = this.sseController.signal
      fetch(BACKEND_BASE + '/api/chat/stream', { signal })
        .then(response => {
          const reader = response.body.getReader()
          const decoder = new TextDecoder()
          let buffer = ''
          const read = () => {
            reader.read().then(({ done, value }) => {
              if (done) {
                setTimeout(() => this.connectSSE(), 3000)
                return
              }
              buffer += decoder.decode(value, { stream: true })
              const lines = buffer.split('\n')
              buffer = lines.pop() || ''
              for (const line of lines) {
                if (line.startsWith('data: ')) {
                  const data = line.slice(6)
                  const obj = this.parseRawObject(data)
                  if (obj && obj.flag === false) continue
                  const content = this.parseContent(data)
                  if (!content) continue
                  this.sending = false
                  const isError = content.startsWith('【ERROR】')
                  const msgType = isError ? 'error' : (obj && obj.type ? obj.type : 'text')
                  const images = msgType === 'image' && content
                    ? (content.includes(',') ? content.split(',') : [content])
                        .map(p => p.trim().startsWith('/') ? BACKEND_BASE + p.trim() : p.trim())
                    : []
                  this.messages.push({
                    role: 'agent',
                    content,
                    type: msgType,
                    images,
                    id: Date.now()
                  })
                  this.scrollToBottom()
                }
              }
              read()
            }).catch(err => {
              if (err.name !== 'AbortError') {
                setTimeout(() => this.connectSSE(), 3000)
              }
            })
          }
          read()
        })
        .catch(err => {
          if (err.name !== 'AbortError') {
            setTimeout(() => this.connectSSE(), 3000)
          }
        })
    },
    closeSSE() {
      if (this.sseController) {
        this.sseController.abort()
        this.sseController = null
      }
    },
    parseRawObject(raw) {
      if (!raw) return null
      try {
        let parsed = JSON.parse(raw)
        if (typeof parsed === 'string') {
          try { parsed = JSON.parse(parsed) } catch (e) { /* not double-encoded */ }
        }
        if (typeof parsed === 'object' && parsed !== null) return parsed
      } catch (e) { /* not JSON */ }
      const typeMatch = raw.match(/'message_type'\s*:\s*'([^']+)'/)
      const flagMatch = raw.match(/'flag'\s*:\s*(True|False)/i)
      if (typeMatch || flagMatch) {
        return {
          message_type: typeMatch ? typeMatch[1] : undefined,
          flag: flagMatch ? flagMatch[1].toLowerCase() === 'true' : undefined
        }
      }
      return null
    },
    parseContent(raw) {
      if (!raw) return ''
      try {
        let parsed = JSON.parse(raw)
        if (typeof parsed === 'string') {
          try { parsed = JSON.parse(parsed) } catch (e) { return parsed }
        }
        if (typeof parsed === 'object' && parsed !== null && parsed.content !== undefined) {
          return String(parsed.content)
        }
        return String(parsed)
      } catch (e) { /* not standard JSON */ }
      const match = raw.match(/'content'\s*:\s*'([\s\S]*?)'\s*(?:,\s*'|\}\s*$)/)
      if (match) {
        return match[1].replace(/\\n/g, '\n')
      }
      return raw
    },
    scrollToBottom() {
      this.$nextTick(() => {
        const el = this.$refs.messagesContainer
        if (el) el.scrollTop = el.scrollHeight
      })
    },
    autoResize(e) {
      const el = e.target
      el.style.height = 'auto'
      el.style.height = Math.min(el.scrollHeight, 120) + 'px'
    },
    resetInputHeight() {
      this.$nextTick(() => {
        const el = this.$refs.inputEl
        if (el) el.style.height = 'auto'
      })
    }
  }
}
</script>

<style scoped>
.chat-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding-top: 48px;
  max-width: 720px;
  margin: 0 auto;
  background: #f7f8fa;
  font-family: 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif;
}

.chat-header {
  padding: 13px 20px;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(12px);
  color: #444;
  text-align: center;
  flex-shrink: 0;
  border-bottom: 1px solid #eef0f2;
}

.agent-name {
  margin: 0;
  font-size: 15px;
  font-weight: 600;
  color: #444;
  letter-spacing: 0.5px;
  cursor: pointer;
  transition: color 0.2s;
  display: inline;
  border-bottom: 1px dashed transparent;
}
.agent-name:hover {
  color: #5b9bd5;
  border-bottom-color: #b8d4ec;
}

.name-input {
  font-size: 15px;
  font-weight: 600;
  color: #444;
  text-align: center;
  border: none;
  border-bottom: 1px dashed #b8d4ec;
  background: transparent;
  outline: none;
  width: 160px;
  font-family: inherit;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  padding: 24px 16px;
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.loading-tip,
.empty-tip {
  text-align: center;
  color: #b0b0b0;
  font-size: 13px;
  padding: 12px 0;
}

.empty-tip {
  margin-top: 60px;
}

.message-row {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  animation: fadeIn 0.25s ease;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(4px); }
  to { opacity: 1; transform: translateY(0); }
}

.message-right {
  flex-direction: row-reverse;
}

.avatar {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 500;
  flex-shrink: 0;
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

.avatar-file-input {
  position: absolute;
  inset: 0;
  opacity: 0;
  cursor: pointer;
}

.message-left .avatar {
  background: #e9ecf0;
  color: #888;
}

.message-right .avatar {
  background: #d6e5f5;
  color: #5b9bd5;
}

.avatar-img {
  width: 100%;
  height: 100%;
  border-radius: 50%;
  object-fit: cover;
}

.message-bubble {
  max-width: 72%;
  padding: 10px 14px;
  border-radius: 14px;
  font-size: 14px;
  line-height: 1.65;
  word-break: break-word;
}

.bubble-user {
  background: #e8f2fc;
  color: #333;
  border-bottom-right-radius: 4px;
}

.bubble-agent {
  background: #fff;
  color: #444;
  border-bottom-left-radius: 4px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.04);
}

.bubble-error {
  background: #fdf2f2;
  color: #c75050;
  border-bottom-left-radius: 4px;
  border: 1px solid #f5d0d0;
}

.message-text {
  margin: 0;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: inherit;
  font-size: inherit;
}

.image-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.chat-image {
  max-width: 200px;
  max-height: 200px;
  border-radius: 8px;
  cursor: pointer;
  object-fit: cover;
  transition: opacity 0.2s;
}
.chat-image:hover {
  opacity: 0.85;
}

/* custom voice player */
.voice-player {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 220px;
  max-width: 260px;
}

.vp-audio {
  display: none;
}

.vp-btn {
  width: 30px;
  height: 30px;
  border-radius: 50%;
  border: 1px dashed #b8d4ec;
  background: transparent;
  color: #6aabd8;
  font-size: 11px;
  cursor: pointer;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.25s;
  line-height: 1;
}
.vp-btn:hover {
  border-color: #5b9bd5;
  color: #4a90c4;
  background: rgba(91, 155, 213, 0.06);
}

.vp-track {
  flex: 1;
  height: 4px;
  background: #e4e8ec;
  border-radius: 4px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
}

.vp-progress {
  height: 100%;
  background: linear-gradient(90deg, #a8cce8, #6aabd8);
  border-radius: 4px;
  transition: width 0.15s linear;
}

.vp-time {
  font-size: 11px;
  color: #aaa;
  white-space: nowrap;
  flex-shrink: 0;
  font-variant-numeric: tabular-nums;
}

/* typing animation */
.typing {
  display: flex;
  gap: 5px;
  padding: 14px 18px;
}

.dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #c0c8d0;
  animation: bounce 1.4s ease-in-out infinite;
}

.dot:nth-child(2) { animation-delay: 0.25s; }
.dot:nth-child(3) { animation-delay: 0.5s; }

@keyframes bounce {
  0%, 60%, 100% { transform: translateY(0); }
  30% { transform: translateY(-4px); }
}

.chat-input-area {
  display: flex;
  align-items: flex-end;
  gap: 10px;
  padding: 12px 16px;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(12px);
  border-top: 1px solid #eef0f2;
  flex-shrink: 0;
}

.chat-input {
  flex: 1;
  resize: none;
  border: 1px solid #e4e8ec;
  border-radius: 10px;
  padding: 10px 14px;
  font-size: 14px;
  line-height: 1.5;
  outline: none;
  max-height: 120px;
  transition: border-color 0.25s, box-shadow 0.25s;
  font-family: inherit;
  background: #f9fafb;
  color: #444;
}

.chat-input:focus {
  border-color: #b8cfe8;
  background: #fff;
  box-shadow: 0 0 0 3px rgba(91, 155, 213, 0.08);
}

.upload-btn {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 1px dashed #b8d4ec;
  background: transparent;
  color: #6aabd8;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.25s;
  position: relative;
}
.upload-btn:hover {
  border-color: #5b9bd5;
  color: #4a90c4;
  background: rgba(91, 155, 213, 0.05);
}
.upload-file-input {
  position: absolute;
  width: 0;
  height: 0;
  opacity: 0;
  pointer-events: none;
}

.send-btn {
  padding: 9px 20px;
  border: 1px dashed #b8d4ec;
  border-radius: 100px;
  background: transparent;
  color: #6aabd8;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  flex-shrink: 0;
  transition: all 0.25s;
}

.send-btn:not(:disabled):hover {
  border-color: #5b9bd5;
  color: #4a90c4;
  background: rgba(91, 155, 213, 0.05);
}

.send-btn:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}

/* Image preview overlay */
.image-overlay {
  position: fixed;
  inset: 0;
  z-index: 1000;
  background: rgba(0, 0, 0, 0.75);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  animation: fadeIn 0.2s ease;
}
.overlay-image {
  max-width: 90vw;
  max-height: 90vh;
  border-radius: 8px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  cursor: default;
}
</style>
