<template>
  <div>
    <nav class="app-nav">
      <button :class="{ active: currentPage === 'chat' }" @click="currentPage = 'chat'">聊天</button>
      <button :class="{ active: currentPage === 'diary' }" @click="currentPage = 'diary'">日记</button>
      <button :class="{ active: currentPage === 'config' }" @click="currentPage = 'config'">配置</button>
    </nav>
    <ChatComponent v-if="currentPage === 'chat'" />
    <DiaryPage v-else-if="currentPage === 'diary'" />
    <ConfigLover v-else />
  </div>
</template>

<script>
import ChatComponent from './components/chat/ChatComponent.vue'
import DiaryPage from './components/diary/DiaryPage.vue'
import ConfigLover from './components/config/ConfigLover.vue'

export default {
  name: 'App',
  components: {
    ChatComponent,
    DiaryPage,
    ConfigLover
  },
  data() {
    return {
      currentPage: 'chat'
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', 'PingFang SC', 'Microsoft YaHei', sans-serif;
  background: #f7f8fa;
  -webkit-font-smoothing: antialiased;
  color: #444;
}

.app-nav {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
  display: flex;
  background: rgba(255, 255, 255, 0.92);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid #eef0f2;
}

.app-nav button {
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

.app-nav button.active {
  color: #5b9bd5;
  border-bottom-color: #5b9bd5;
}

.app-nav button:hover:not(.active) {
  color: #666;
}
</style>
