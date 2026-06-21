<template>
  <div class="memory-config">
    <!-- 三级导航 - 中期记忆/长期记忆 -->
    <div class="level3-tabs">
      <button
        :class="{ active: memoryType === 'mid' }"
        @click="memoryType = 'mid'"
      >中期记忆</button>
      <button
        :class="{ active: memoryType === 'long' }"
        @click="memoryType = 'long'"
      >长期记忆</button>
    </div>

    <div class="memory-content">
      <div v-if="loading" class="loading-tip">加载中...</div>
      <template v-else>
        <div v-if="message" :class="['message-bar', messageType]">{{ message }}</div>

        <!-- 操作栏 -->
        <div class="action-bar">
          <span class="memory-count">共 {{ memories.length }} 条记忆</span>
          <div class="action-buttons">
            <button class="btn-ghost" @click="refreshList">刷新</button>
            <button
              class="btn-ghost-danger"
              :disabled="selectedIds.length === 0"
              @click="deleteSelected"
            >删除选中 ({{ selectedIds.length }})</button>
          </div>
        </div>

        <!-- 记忆列表 -->
        <div class="memory-list" v-if="memories.length > 0">
          <div class="memory-header">
            <label class="checkbox-label">
              <input type="checkbox" :checked="isAllSelected" @change="toggleSelectAll" />
              全选
            </label>
            <span class="header-content">内容</span>
            <span class="header-time">创建时间</span>
            <span class="header-action">操作</span>
          </div>
          <div v-for="item in memories" :key="item.id" class="memory-item">
            <label class="checkbox-label">
              <input type="checkbox" :value="item.id" v-model="selectedIds" />
            </label>
            <div class="memory-content-text">{{ item.content }}</div>
            <span class="memory-time">{{ formatTime(item.create_time) }}</span>
            <div class="memory-action">
              <button class="btn-del-sm" @click="deleteSingle(item.id)">删除</button>
            </div>
          </div>
        </div>
        <div v-else class="empty-tip">暂无记忆数据</div>
      </template>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ConfigMemory',
  data() {
    return {
      memoryType: 'mid', // 'mid' 或 'long'
      memories: [],
      loading: false,
      message: '',
      messageType: 'success',
      selectedIds: []
    }
  },
  computed: {
    isAllSelected() {
      return this.memories.length > 0 && this.selectedIds.length === this.memories.length
    }
  },
  watch: {
    memoryType() {
      this.selectedIds = []
      this.loadMemories()
    }
  },
  mounted() {
    this.loadMemories()
  },
  methods: {
    async loadMemories() {
      this.loading = true
      this.message = ''
      try {
        const url = this.memoryType === 'mid'
          ? '/api/memory/mid/list'
          : '/api/memory/long/list'
        const res = await fetch(url, { cache: 'no-cache' })
        if (res.ok) {
          this.memories = await res.json()
        } else {
          this.showMessage('加载失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('加载失败: ' + e.message, 'error')
      } finally {
        this.loading = false
      }
    },
    refreshList() {
      this.selectedIds = []
      this.loadMemories()
    },
    toggleSelectAll() {
      if (this.isAllSelected) {
        this.selectedIds = []
      } else {
        this.selectedIds = this.memories.map(m => m.id)
      }
    },
    async deleteSingle(id) {
      if (!confirm('确定删除这条记忆？')) return
      await this.deleteMemories([id])
    },
    async deleteSelected() {
      if (this.selectedIds.length === 0) return
      if (!confirm(`确定删除选中的 ${this.selectedIds.length} 条记忆？`)) return
      await this.deleteMemories([...this.selectedIds])
    },
    async deleteMemories(ids) {
      try {
        const url = this.memoryType === 'mid'
          ? '/api/memory/mid/delete'
          : '/api/memory/long/delete'
        const res = await fetch(url, {
          method: 'DELETE',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(ids)
        })
        if (res.ok) {
          this.showMessage(`成功删除 ${ids.length} 条记忆`, 'success')
          this.selectedIds = this.selectedIds.filter(id => !ids.includes(id))
          await this.loadMemories()
        } else {
          this.showMessage('删除失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('删除失败: ' + e.message, 'error')
      }
    },
    formatTime(timeStr) {
      if (!timeStr) return ''
      const d = new Date(timeStr)
      return d.toLocaleString('zh-CN', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        minute: '2-digit'
      })
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
.memory-config {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

/* 三级导航 */
.level3-tabs {
  display: flex;
  background: rgba(250, 251, 252, 0.9);
  border-bottom: 1px solid #eef0f2;
  flex-shrink: 0;
}
.level3-tabs button {
  flex: 1;
  padding: 9px 0;
  border: none;
  background: none;
  font-size: 13px;
  font-weight: 500;
  color: #aaa;
  cursor: pointer;
  border-bottom: 2px solid transparent;
  transition: color 0.25s, border-color 0.25s;
}
.level3-tabs button.active {
  color: #5b9bd5;
  border-bottom-color: #5b9bd5;
  background: rgba(255, 255, 255, 0.6);
}
.level3-tabs button:hover:not(.active) {
  color: #777;
}

.memory-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.loading-tip { text-align: center; color: #b0b0b0; padding: 40px; }
.empty-tip { text-align: center; color: #b0b0b0; padding: 40px; }

.message-bar { padding: 8px 16px; font-size: 13px; text-align: center; flex-shrink: 0; }
.message-bar.success { background: #f2f9f2; color: #4a9a5b; }
.message-bar.error { background: #fdf2f2; color: #c75050; }

/* 操作栏 */
.action-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 16px;
  background: #fafbfc;
  border-bottom: 1px solid #eef0f2;
  flex-shrink: 0;
}
.memory-count { font-size: 12px; color: #aaa; }
.action-buttons { display: flex; gap: 8px; }

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
.btn-ghost-danger:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}
.btn-ghost-danger:disabled:hover {
  background: transparent;
}

/* 记忆列表 */
.memory-list {
  flex: 1;
  overflow-y: auto;
  padding: 0 16px 16px;
}
.memory-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  background: #fafbfc;
  border-radius: 10px;
  margin: 12px 0 8px;
  font-size: 12px;
  font-weight: 600;
  color: #aaa;
}
.memory-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  border: 1px solid #eef0f2;
  border-radius: 10px;
  margin-bottom: 8px;
  background: #fff;
  transition: border-color 0.25s, box-shadow 0.25s;
}
.memory-item:hover {
  border-color: #d8dfe6;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.03);
}

.checkbox-label {
  display: flex;
  align-items: center;
  cursor: pointer;
  flex-shrink: 0;
}
.checkbox-label input[type="checkbox"] {
  accent-color: #8bbde6;
  width: 15px;
  height: 15px;
}

.memory-content-text {
  flex: 1;
  font-size: 13px;
  color: #444;
  line-height: 1.5;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}
.header-content {
  flex: 1;
  padding-left: 28px;
}

.memory-time {
  font-size: 12px;
  color: #b0b0b0;
  flex-shrink: 0;
  min-width: 140px;
  text-align: center;
}
.header-time {
  min-width: 140px;
  text-align: center;
}

.memory-action {
  flex-shrink: 0;
  min-width: 60px;
  text-align: center;
}
.header-action {
  min-width: 60px;
  text-align: center;
}

.btn-del-sm {
  padding: 3px 10px;
  border: 1px dashed #f0b3be;
  border-radius: 100px;
  background: none;
  color: #d4768a;
  font-size: 11px;
  cursor: pointer;
  transition: all 0.25s;
}
.btn-del-sm:hover { background: rgba(212, 118, 138, 0.06); border-color: #d4768a; }
</style>
