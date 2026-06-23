# Magellan Lovers Frontend

基于 Vue 3 的 AI 伴侣应用前端，提供实时聊天、配置管理和记忆管理功能。

后端项目：[magellan_lover](https://github.com/magellan41/magellan_lover)

## 技术栈

- Vue 3 (Options API)
- Vue CLI 5
- SSE (Server-Sent Events) 实时通信
- 原生 fetch API（无额外网络库）

## 项目结构

```
src/
├── App.vue                          # 主页面路由（聊天 / 配置）
├── main.js
└── components/
    ├── chat/
    │   └── ChatComponent.vue        # 聊天页面（SSE、分页、语音播放、图片消息、遮罩预览）
    └── config/
        ├── ConfigLover.vue           # 配置管理容器（二级导航）
        ├── ConfigAgent.vue           # Agent 配置（LLM 平台 + 模型选择）
        ├── ConfigMemory.vue          # 记忆管理（中期/长期记忆）
        └── ConfigMore.vue            # 更多设置（语音、日程描述、图片生成工具）
```

## 功能模块

### 聊天页面
- SSE 实时接收 AI 回复
- 分页加载历史记录（滚动到顶部自动加载更早消息）
- 语音消息自定义播放器（播放/暂停/进度/跳转）
- 图片消息展示（逗号分隔多图网格 + 本页遮罩层预览）
- 动态头像（点击上传更换）
- 动态 Agent 名称（点击编辑）

### 配置管理
- **伴侣配置**：Soul / User 两个 Markdown 配置文件的编辑与预览
- **Agent 配置**：LLM 平台管理（base_url、api_key、模型列表）+ Agent 模型选择（chat / compact / memory / story）
- **记忆管理**：中期 / 长期记忆的查看与删除
- **更多设置**：语音配置（开关 + API Key + 生成器）、日程描述编辑、图片生成工具配置（平台下拉 + API Key + 模型 + 基准图片上传）

## 快速开始

```bash
# 安装依赖
npm install

# 启动开发服务器（默认 http://localhost:8080）
npm run serve

# 构建生产版本
npm run build

# 代码检查
npm run lint
```

## 后端接口

开发服务器自动将 `/api` 代理到 `http://127.0.0.1:8000`。

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/chat/list/{min_id}` | GET | 获取聊天记录（分页） |
| `/api/chat/send` | POST | 发送消息 |
| `/api/chat/image` | POST | 上传图片 |
| `/api/chat/stream` | GET | SSE 实时消息推送 |
| `/api/agent/env/get/{key}` | GET | 获取环境配置 |
| `/api/agent/env/set` | POST | 设置环境配置 |
| `/api/agent/config/get` | GET | 获取 Agent 配置 |
| `/api/agent/config/set` | POST | 设置 Agent 配置 |
| `/api/agent/voice/get` | GET | 获取语音配置 |
| `/api/agent/voice/set` | POST | 设置语音配置 |
| `/api/agent/schedule_description/get` | GET | 获取日程描述 |
| `/api/agent/schedule_description/set` | POST | 设置日程描述 |
| `/api/agent/image_generator/platform/get` | GET | 获取图片生成平台列表 |
| `/api/lover/config/show/{config}` | GET | 获取伴侣配置 |
| `/api/lover/config/set/{config}` | POST | 设置伴侣配置 |
| `/api/file/uploads/avatar/{type}` | POST | 上传头像 |
| `/api/file/uploads/character_image` | POST | 上传基准图片 |
| `/api/memory/{type}/list` | GET | 获取记忆列表 |
| `/api/memory/{type}/delete` | DELETE | 删除记忆 |

> SSE 接口直连 `http://localhost:8000`（webpack-dev-server 代理不支持 SSE 流转发）。
