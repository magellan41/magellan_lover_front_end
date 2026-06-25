<template>
  <div class="diary-page">
    <div v-if="loading" class="loading-tip">加载中...</div>
    <template v-else>
      <div class="diary-layout">
        <!-- 左侧标题列表 -->
        <div class="diary-sidebar">
          <div class="sidebar-header">
            <h3>日记列表</h3>
            <button class="btn-refresh" @click="loadTitles(-1)">刷新</button>
          </div>
          <div class="title-list" ref="titleList" @scroll="onSidebarScroll">
            <div
              v-for="(title, i) in titles"
              :key="i"
              :class="['title-item', { active: currentTitle === title }]"
              @click="selectTitle(title)"
            >
              {{ title }}
            </div>
            <div v-if="titles.length === 0" class="empty-tip">暂无日记</div>
            <div v-if="loadingMore" class="loading-more">加载中...</div>
          </div>
        </div>

        <!-- 右侧内容区 -->
        <div class="diary-content">
          <div v-if="!currentTitle" class="empty-tip">请从左侧选择一篇日记</div>
          <template v-else>
            <div class="content-header">
              <h2>{{ diaryContent.title || currentTitle }}</h2>
            </div>
            <div class="content-body" v-html="renderContent(diaryContent.content)"></div>
          </template>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
export default {
  name: 'DiaryPage',
  data() {
    return {
      loading: false,
      loadingMore: false,
      titles: [],
      currentTitle: '',
      diaryContent: {},
      canLoadMore: true
    }
  },
  mounted() {
    this.loadTitles(-1)
  },
  methods: {
    async loadTitles(miniId) {
      if (miniId === -1) {
        this.loading = true
        this.titles = []
        this.canLoadMore = true
      } else {
        this.loadingMore = true
      }
      try {
        const res = await fetch(`/api/agent/diary/list/${miniId}`, { cache: 'no-cache' })
        if (res.ok) {
          const data = await res.json()
          const list = data.data || []
          if (miniId === -1) {
            this.titles = list
          } else {
            this.titles.push(...list)
          }
          if (list.length === 0) this.canLoadMore = false
        }
      } catch (e) {
        /* 静默处理 */
      } finally {
        this.loading = false
        this.loadingMore = false
      }
    },
    onSidebarScroll(e) {
      const el = e.target
      if (el.scrollTop + el.clientHeight >= el.scrollHeight - 50 && this.canLoadMore && !this.loadingMore) {
        const lastTitle = this.titles[this.titles.length - 1]
        if (lastTitle) this.loadTitles(lastTitle)
      }
    },
    async selectTitle(title) {
      if (this.currentTitle === title) return
      this.currentTitle = title
      this.diaryContent = {}
      try {
        const res = await fetch(`/api/agent/diary/get/${encodeURIComponent(title)}`, { cache: 'no-cache' })
        if (res.ok) {
          this.diaryContent = await res.json()
        }
      } catch (e) {
        /* 静默处理 */
      }
    },
    renderContent(text) {
      if (!text) return ''
      return text
        .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
        .replace(/\n\n/g, '<br><br>').replace(/\n/g, '<br>')
    }
  }
}
</script>

<style scoped>
.diary-page {
  height: 100vh;
  padding-top: 50px;
  display: flex;
  flex-direction: column;
  max-width: 900px;
  margin: 0 auto;
  background: #f7f8fa;
}
.loading-tip { text-align: center; color: #b0b0b0; padding: 40px; }
.empty-tip { text-align: center; color: #bbb; padding: 40px; font-size: 14px; }

.diary-layout {
  flex: 1;
  display: flex;
  overflow: hidden;
}

/* 左侧标题栏 */
.diary-sidebar {
  width: 240px;
  background: #fff;
  border-right: 1px solid #eef0f2;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
}
.sidebar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 16px;
  border-bottom: 1px solid #eef0f2;
  background: #fafbfc;
}
.sidebar-header h3 {
  margin: 0;
  font-size: 14px;
  font-weight: 600;
  color: #444;
}
.btn-refresh {
  padding: 3px 10px;
  border: 1px dashed #b8d4ec;
  border-radius: 100px;
  font-size: 12px;
  cursor: pointer;
  background: transparent;
  color: #6aabd8;
  transition: all 0.25s;
}
.btn-refresh:hover {
  border-color: #5b9bd5;
  background: rgba(91, 155, 213, 0.06);
}

.title-list {
  flex: 1;
  overflow-y: auto;
  padding: 8px 0;
}
.title-item {
  padding: 10px 16px;
  font-size: 13px;
  color: #555;
  cursor: pointer;
  border-left: 3px solid transparent;
  transition: all 0.2s;
}
.title-item:hover {
  background: #f5f7fa;
}
.title-item.active {
  background: #f0f6fc;
  color: #5b9bd5;
  border-left-color: #5b9bd5;
  font-weight: 500;
}
.loading-more {
  text-align: center;
  padding: 10px;
  font-size: 12px;
  color: #bbb;
}

/* 右侧内容区 */
.diary-content {
  flex: 1;
  overflow-y: auto;
  background: #fff;
}
.content-header {
  padding: 20px 24px 0;
}
.content-header h2 {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  color: #333;
}
.content-body {
  padding: 16px 24px 24px;
  font-size: 15px;
  line-height: 1.85;
  color: #444;
  word-break: break-word;
}
</style>
