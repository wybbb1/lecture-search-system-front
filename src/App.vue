<template>
  <div class="app-container">
    <div class="right-panel">
      <SearchPartInterface />
    </div>

    <Transition name="fade-scale">
      <div v-if="showChat" 
           class="ai-chat-overlay"
           :style="overlayStyle"
           @click.self="hideChat"> <div class="ai-chat-content">
          <ChatInterface @close-chat="hideChat" />
        </div>
      </div>
    </Transition>

    <div v-if="!showChat" 
         class="chat-toggle-button" 
         @click="openChatFromButton"
         ref="chatToggleButton">
      <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V3a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path>
      </svg>
      <span>AI 助手</span>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, nextTick } from 'vue';
import ChatInterface from './components/ChatInterface.vue';
import SearchPartInterface from './components/SearchPartInterface.vue';

const showChat = ref(false);
const chatToggleButton = ref<HTMLElement | null>(null);
const overlayStyle = reactive({
  '--overlay-start-x': '0px',
  '--overlay-start-y': '0px',
  '--overlay-width': '0px',
  '--overlay-height': '0px',
});

// 计算扩散起始位置和大小
const openChatFromButton = async () => {
  if (chatToggleButton.value) {
    const rect = chatToggleButton.value.getBoundingClientRect();
    overlayStyle['--overlay-start-x'] = `${rect.left + rect.width / 2}px`;
    overlayStyle['--overlay-start-y'] = `${rect.top + rect.height / 2}px`;
    overlayStyle['--overlay-width'] = `${rect.width}px`;
    overlayStyle['--overlay-height'] = `${rect.height}px`;
  }
  showChat.value = true;
};

const hideChat = () => {
  showChat.value = false;
};
</script>

<style>
/* 全局样式 */
body, html, #app {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
  overflow: hidden;
  font-family: -apple-system, BlinkMacMacFont, 'Segoe UI', 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', 'Helvetica Neue', Helvetica, Arial, sans-serif;
  background-color: #f0f2f5;
}

.app-container {
  display: flex;
  height: 100vh;
  width: 100vw;
  overflow: hidden;
}

.right-panel {
  flex: 1; /* SearchPartInterface 占据全部宽度 */
  display: flex;
  flex-direction: column;
  height: 100%;
  overflow: hidden;
}

/* AI 助手扩散覆盖层样式 */
.ai-chat-overlay {
  position: fixed;
  /* 调整大小和位置 */
  width: 40vw; /* 占据视口宽度的一半 */
  height: 50vh; /* 占据视口高度的一半 */
  right: 0; /* 从右侧开始定位 */
  top: 40vh; /* 距离顶部 1/4 视口高度 (100vh / 4) */
  box-sizing: border-box; /* 包含 padding 和 border 在 width/height 内 */

  display: flex;
  justify-content: flex-end; /* ChatInterface 内容靠右 */
  align-items: center; /* 垂直居中 */
  background-color: rgba(255, 255, 255, 0.6); /* 半透明背景 */
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  z-index: 100; /* 确保在 SearchPartInterface 上方 */
  box-shadow: -4px 0 15px rgba(0, 0, 0, 0.1); /* 右侧阴影 */
  overflow: hidden; /* 防止内容溢出扩散边界 */
  border-radius: 15px; /* 添加圆角效果 */
}

.ai-chat-content {
  width: 100%; /* ChatInterface 占据覆盖层宽度 */
  height: 100%; /* ChatInterface 占据覆盖层高度 */
  display: flex;
  flex-direction: column;
}


/* 唤出 AI 助手按钮样式 */
.chat-toggle-button {
  position: fixed;
  right: 20px;
  bottom: calc(100vh / 3); /* 距离底部 1/3 视口高度 */
  background: linear-gradient(45deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 12px 18px;
  border-radius: 12px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
  cursor: pointer;
  z-index: 90;
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 1rem;
  font-weight: 500;
  transition: all 0.3s ease;
}

.chat-toggle-button:hover {
  background: linear-gradient(45deg, #536ee9 0%, #63368e 100%);
  transform: translateY(-3px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
}

.chat-toggle-button svg {
  width: 20px;
  height: 20px;
}

/* 媒体查询：小屏幕下的布局调整 */
@media (max-width: 1024px) {
  .app-container {
    flex-direction: column;
    overflow-y: auto;
  }

  .right-panel {
    height: auto;
    flex: none;
    width: 100%;
    border-bottom: 1px solid #e2e8f0;
  }

  .ai-chat-overlay {
    position: relative; /* 小屏幕下变为相对定位 */
    width: 100%;
    height: 50vh; /* 保持半屏高 */
    min-height: 300px; /* 最小高度，防止过小 */
    background-color: #f8fafc; /* 恢复正常背景色 */
    backdrop-filter: none; /* 移除毛玻璃，避免性能问题 */
    -webkit-backdrop-filter: none;
    box-shadow: none;
    top: auto;
    right: auto;
    border-radius: 0; /* 移除圆角 */
    justify-content: center; /* 内容居中 */
  }

  .ai-chat-content {
    width: 100%;
    height: 100%;
  }

  /* 移除小屏幕上的扩散动画 */
  .fade-scale-enter-active,
  .fade-scale-leave-active {
    transition: none;
  }
  .fade-scale-enter-from,
  .fade-scale-leave-to {
    opacity: 0;
    clip-path: none; /* 移除 clip-path */
    width: 100%; /* 确保小屏幕下宽度始终为 100% */
    height: 50vh;
    top: auto;
    right: auto;
    transform: none;
  }
  .fade-scale-enter-to,
  .fade-scale-leave-from {
    opacity: 1;
    clip-path: none; /* 移除 clip-path */
    width: 100%;
    height: 50vh;
    top: auto;
    right: auto;
    transform: none;
  }

  .chat-toggle-button {
    display: none; /* 小屏幕时隐藏浮动按钮 */
  }
}
</style>