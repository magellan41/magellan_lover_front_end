<template>
  <div class="config-task">
    <div v-if="loading" class="loading-tip">加载中...</div>
    <template v-else>
      <div v-if="message" :class="['message-bar', messageType]">{{ message }}</div>

      <div class="task-list" v-if="tasks.length">
        <div v-for="task in tasks" :key="task.id" class="task-item">
          <div class="task-info">
            <div class="task-name">{{ task.task_name || '未命名任务' }}</div>
            <div class="task-prompt">{{ task.prompt }}</div>
            <div class="task-meta">
              <span class="task-status" :class="'status-' + task.status">
                {{ statusLabel(task.status) }}
              </span>
              <span class="task-time">
                触发: {{ formatTime(task.trigger_time) }}
              </span>
            </div>
          </div>
          <button class="task-del" @click="deleteTask(task)">删除</button>
        </div>
      </div>
      <div v-else class="empty-tip">暂无任务</div>
    </template>
  </div>
</template>

<script>
export default {
  name: 'ConfigTask',
  data() {
    return {
      loading: false,
      tasks: [],
      message: '',
      messageType: 'success'
    }
  },
  mounted() {
    this.loadTasks()
  },
  methods: {
    async loadTasks() {
      this.loading = true
      try {
        const res = await fetch('/api/agent/remind/list', { cache: 'no-cache' })
        if (res.ok) {
          const json = await res.json()
          const list = json.data || json
          this.tasks = Array.isArray(list) ? list : []
        } else {
          this.showMessage('加载任务列表失败: ' + res.status, 'error')
        }
      } catch (e) {
        this.showMessage('加载任务列表失败', 'error')
      } finally {
        this.loading = false
      }
    },
    async deleteTask(task) {
      if (!confirm(`确定删除任务「${task.task_name || '未命名'}」？`)) return
      try {
        const res = await fetch('/api/agent/remind/delete', {
          method: 'DELETE',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify([task.id])
        })
        if (res.ok) {
          this.tasks = this.tasks.filter(t => t.id !== task.id)
          this.showMessage('任务已删除', 'success')
        } else {
          const err = await res.json().catch(() => ({}))
          this.showMessage('删除失败: ' + (err.detail || res.status), 'error')
        }
      } catch (e) {
        this.showMessage('删除失败: 网络错误', 'error')
      }
    },
    statusLabel(status) {
      const map = { 1: '待执行', 2: '已执行', 0: '已取消' }
      return map[status] || '未知'
    },
    formatTime(t) {
      if (!t) return '-'
      try { return new Date(t).toLocaleString('zh-CN') } catch (e) { return t }
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
.config-task {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.loading-tip { text-align: center; color: #b0b0b0; padding: 40px; }
.empty-tip { text-align: center; color: #bbb; padding: 40px; font-size: 14px; }

.message-bar { padding: 8px 16px; font-size: 13px; text-align: center; flex-shrink: 0; }
.message-bar.success { background: #f2f9f2; color: #4a9a5b; }
.message-bar.error { background: #fdf2f2; color: #c75050; }

.task-list {
  flex: 1;
  overflow-y: auto;
  padding: 12px 16px;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  background: #fff;
  border: 1px solid #eef0f2;
  border-radius: 10px;
  margin-bottom: 10px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.03);
}

.task-info {
  flex: 1;
  min-width: 0;
}

.task-name {
  font-size: 14px;
  font-weight: 600;
  color: #444;
  margin-bottom: 4px;
}

.task-prompt {
  font-size: 13px;
  color: #777;
  line-height: 1.5;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.task-meta {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-top: 6px;
}

.task-status {
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 100px;
  font-weight: 500;
}
.status-1 { background: #fff3e0; color: #e67e22; }
.status-2 { background: #e8f5e9; color: #4a9a5b; }
.status-0 { background: #f5f5f5; color: #999; }

.task-time {
  font-size: 12px;
  color: #aaa;
}

.task-del {
  padding: 4px 12px;
  border: 1px dashed #f0b3be;
  border-radius: 100px;
  font-size: 12px;
  cursor: pointer;
  background: transparent;
  color: #d4768a;
  transition: all 0.25s;
  flex-shrink: 0;
}
.task-del:hover {
  background: #fdf2f2;
  border-color: #d4768a;
}
</style>
