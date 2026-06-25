<template>
  <div class="config-container">
    <div class="config-header">
      <h2>配置管理</h2>
    </div>

    <!-- 一级导航 -->
    <div class="level1-tabs">
      <button
        :class="{ active: level1 === 'lover' }"
        @click="level1 = 'lover'"
      >伴侣配置</button>
      <button
        :class="{ active: level1 === 'agent' }"
        @click="level1 = 'agent'"
      >Agent 配置</button>
      <button
        :class="{ active: level1 === 'memory' }"
        @click="level1 = 'memory'"
      >记忆</button>
      <button
        :class="{ active: level1 === 'task' }"
        @click="level1 = 'task'"
      >任务管理</button>
      <button
        :class="{ active: level1 === 'more' }"
        @click="level1 = 'more'"
      >更多配置</button>
    </div>

    <!-- 伴侣配置 -->
    <template v-if="level1 === 'lover'">
      <div class="level2-tabs">
        <button
          v-for="tab in loverTabs"
          :key="tab.key"
          :class="{ active: loverCurrentTab === tab.key }"
          @click="switchLoverTab(tab.key)"
        >{{ tab.label }}</button>
      </div>
      <div class="config-content">
        <div v-if="loverLoading" class="loading-tip">加载中...</div>
        <template v-else>
          <div class="editor-toolbar">
            <span class="file-name">{{ loverCurrentTab }}.md</span>
            <div class="toolbar-actions">
              <button class="btn-ghost" @click="loverPreview = !loverPreview">
                {{ loverPreview ? '编辑' : '预览' }}
              </button>
              <button class="btn-ghost-primary" :disabled="loverSaving || !loverHasChanges" @click="saveLoverConfig">
                {{ loverSaving ? '保存中...' : '保存' }}
              </button>
            </div>
          </div>
          <div v-if="loverMessage" :class="['message-bar', loverMessageType]">{{ loverMessage }}</div>
          <textarea
            v-if="!loverPreview"
            v-model="loverContent"
            class="config-editor"
            placeholder="Markdown 配置内容..."
            spellcheck="false"
          ></textarea>
          <div v-else class="config-preview" v-html="renderedMarkdown"></div>
        </template>
      </div>
    </template>

    <!-- Agent 配置 -->
    <ConfigAgent v-else-if="level1 === 'agent'" />

    <!-- 记忆管理 -->
    <ConfigMemory v-else-if="level1 === 'memory'" />

    <!-- 更多配置 -->
    <ConfigMore v-else-if="level1 === 'more'" />

    <!-- 任务管理 -->
    <ConfigTask v-else />
  </div>
</template>

<script>
import ConfigAgent from './ConfigAgent.vue'
import ConfigMemory from './ConfigMemory.vue'
import ConfigMore from './ConfigMore.vue'
import ConfigTask from './ConfigTask.vue'

export default {
  name: 'ConfigLover',
  components: { ConfigAgent, ConfigMemory, ConfigMore, ConfigTask },
  data() {
    return {
      level1: 'lover',
      // 伴侣配置
      loverTabs: [
        { key: 'soul', label: 'Soul 灵魂配置' },
        { key: 'user', label: 'User 用户配置' }
      ],
      loverCurrentTab: 'soul',
      loverContent: '',
      loverOriginal: '',
      loverLoading: false,
      loverSaving: false,
      loverPreview: false,
      loverMessage: '',
      loverMessageType: 'success'
    }
  },
  computed: {
    loverHasChanges() {
      return this.loverContent !== this.loverOriginal
    },
    renderedMarkdown() {
      return this.loverContent
        .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
        .replace(/^### (.+)$/gm, '<h3>$1</h3>')
        .replace(/^## (.+)$/gm, '<h2>$1</h2>')
        .replace(/^# (.+)$/gm, '<h1>$1</h1>')
        .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
        .replace(/\*(.+?)\*/g, '<em>$1</em>')
        .replace(/`(.+?)`/g, '<code>$1</code>')
        .replace(/^- (.+)$/gm, '<li>$1</li>')
        .replace(/\n\n/g, '<br><br>').replace(/\n/g, '<br>')
    }
  },
  mounted() {
    this.loadLoverConfig()
  },
  methods: {
    switchLoverTab(key) {
      if (this.loverCurrentTab === key) return
      if (this.loverHasChanges && !confirm('有未保存的修改，是否放弃？')) return
      this.loverCurrentTab = key
      this.loverPreview = false
      this.loverMessage = ''
      this.loadLoverConfig()
    },
    async loadLoverConfig() {
      this.loverLoading = true
      try {
        const res = await fetch(`/api/lover/config/show/${this.loverCurrentTab}`, { cache: 'no-cache' })
        const text = await res.text()
        this.loverContent = text.startsWith('"') ? JSON.parse(text) : text
        this.loverOriginal = this.loverContent
      } catch (e) {
        this.showLoverMessage('加载配置失败', 'error')
      } finally {
        this.loverLoading = false
      }
    },
    async saveLoverConfig() {
      if (!this.loverHasChanges || this.loverSaving) return
      this.loverSaving = true
      try {
        const res = await fetch(`/api/lover/config/set/${this.loverCurrentTab}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(this.loverContent)
        })
        if (res.ok) {
          this.loverOriginal = this.loverContent
          this.showLoverMessage('保存成功！', 'success')
        } else {
          this.showLoverMessage('保存失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showLoverMessage('保存失败: 网络错误', 'error')
      } finally {
        this.loverSaving = false
      }
    },
    showLoverMessage(text, type) {
      this.loverMessage = text
      this.loverMessageType = type
      setTimeout(() => { this.loverMessage = '' }, 3000)
    }
  }
}
</script>

<style scoped>
.config-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  padding-top: 50px;
  max-width: 900px;
  margin: 0 auto;
  background: #f7f8fa;
  font-family: 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif;
}

/* 一级导航 */
.level1-tabs {
  display: flex;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid #eef0f2;
  flex-shrink: 0;
}
.level1-tabs button {
  flex: 1;
  padding: 13px 0;
  border: none;
  background: none;
  font-size: 14px;
  font-weight: 500;
  color: #aaa;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: color 0.25s, border-color 0.25s;
}
.level1-tabs button.active {
  color: #5b9bd5;
  border-bottom-color: #5b9bd5;
}
.level1-tabs button:hover:not(.active) {
  color: #777;
}

/* 二级导航 */
.level2-tabs {
  display: flex;
  background: rgba(250, 251, 252, 0.9);
  border-bottom: 1px solid #eef0f2;
  flex-shrink: 0;
}
.level2-tabs button {
  flex: 1;
  padding: 10px 0;
  border: none;
  background: none;
  font-size: 13px;
  font-weight: 500;
  color: #aaa;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: color 0.25s, border-color 0.25s;
}
.level2-tabs button.active {
  color: #5b9bd5;
  border-bottom-color: #8bbde6;
}
.level2-tabs button:hover:not(.active) {
  color: #777;
}

.config-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: #fff;
  border-radius: 0;
}
.loading-tip { text-align: center; color: #b0b0b0; padding: 40px; }

/* 工具栏 */
.editor-toolbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 16px;
  background: #fafbfc;
  border-bottom: 1px solid #eef0f2;
  flex-shrink: 0;
}
.file-name { font-size: 12px; color: #b0b0b0; font-family: monospace; }
.toolbar-actions { display: flex; gap: 8px; }

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
  padding: 4px 14px;
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
.btn-ghost:disabled,
.btn-ghost-primary:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
.btn-ghost-primary:disabled:hover {
  background: transparent;
}

.message-bar { padding: 8px 16px; font-size: 13px; text-align: center; flex-shrink: 0; border-radius: 0; }
.message-bar.success { background: #f2f9f2; color: #4a9a5b; }
.message-bar.error { background: #fdf2f2; color: #c75050; }

.config-editor {
  flex: 1;
  resize: none;
  border: none;
  padding: 16px;
  font-size: 14px;
  line-height: 1.75;
  font-family: 'Cascadia Code', 'Fira Code', 'Consolas', monospace;
  outline: none;
  background: #f9fafb;
  color: #444;
}
.config-preview {
  flex: 1;
  overflow-y: auto;
  padding: 16px 20px;
  font-size: 14px;
  line-height: 1.8;
  background: #fff;
  color: #444;
}
.config-preview :deep(h1) { font-size: 22px; margin: 16px 0 8px; }
.config-preview :deep(h2) { font-size: 18px; margin: 14px 0 6px; }
.config-preview :deep(h3) { font-size: 16px; margin: 12px 0 4px; }
.config-preview :deep(code) { background: #f2f3f5; padding: 2px 6px; border-radius: 4px; font-size: 13px; }
.config-preview :deep(li) { margin-left: 20px; }
</style>
