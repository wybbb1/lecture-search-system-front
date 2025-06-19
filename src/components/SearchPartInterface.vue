<template>
  <div class="search-container">
    <header class="search-header">
      <h1 class="search-title">讲座检索</h1>
    </header>

    <div class="search-input-group">
      <input
        type="text"
        v-model="query"
        placeholder="请输入讲座关键词..."
        @keyup.enter="performSearch"
        class="search-input"
      />
      <button @click="performSearch" class="search-button">搜索</button>
    </div>

    <div v-if="spellingSuggestion" class="spelling-suggestion-box">
      <p>
        您是不是想搜：
        <span class="suggestion-text" @click="applySuggestion(spellingSuggestion)">
          {{ spellingSuggestion }}
        </span>
        <span class="suggestion-tip"> (点击使用)</span>
      </p>
    </div>

    <div v-if="searchResults.length > 0" class="search-results">
      <h2>搜索结果：</h2>
      <ul class="result-list">
        <li v-for="lecture in searchResults" :key="lecture.id" class="result-item">
          <a href="#" @click.prevent="toggleLectureDetail(lecture)" class="lecture-link">
            <h3 class="lecture-title" v-html="highlightMatches(lecture.title, query)"></h3>
            <div
              class="lecture-preview"
              :class="{ 'expanded': lecture.isExpanded }"
              :style="{ maxHeight: lecture.isExpanded ? 'none' : '60px' }"
            >
              <p v-html="highlightMatches(lecture.content, query)"></p>
            </div>
            <div class="read-more-toggle">
              {{ lecture.isExpanded ? '收起 ▲' : '查看更多 ▼' }}
            </div>
          </a>
        </li>
      </ul>
    </div>

    <div v-else-if="searched && searchResults.length === 0" class="no-results">
      <div class="no-results-icon">
        <svg xmlns="http://www.w3.org/2000/svg" width="60" height="60" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="feather feather-search-off">
          <path d="M10 10l5 5"></path>
          <path d="M15 10l-5 5"></path>
          <circle cx="11" cy="11" r="8"></circle>
          <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
        </svg>
      </div>
      <p class="no-results-message">抱歉，未能找到相关的讲座信息。</p>
      <p class="no-results-tip">请尝试更换关键词，或缩短查询内容。</p>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue';
import axios from 'axios';

interface LectureDocumentVO {
  id: number;
  title: string;
  content: string;
  isExpanded?: boolean;
}

interface BackendResponse {
  success: boolean;
  errorMsg: string | null;
  data: LectureDocumentVO[] | null;
  total: number | null;
}

// 新增：拼写建议接口响应类型
interface SpellingAdviceResponse {
  success: boolean;
  errorMsg: string | null;
  data: string | null; // 拼写建议文本
}

const query = ref<string>('');
const searchResults = ref<LectureDocumentVO[]>([]);
const searched = ref<boolean>(false);
const spellingSuggestion = ref<string | null>(null); // 新增：拼写建议

/**
 * 高亮匹配词项
 * @param text 原始文本
 * @param searchQuery 用户查询词
 * @returns 带有高亮标记的 HTML 字符串
 */
const highlightMatches = (text: string, searchQuery: string): string => {
  if (!searchQuery || !text) {
    return text;
  }

  const queryTerms = searchQuery.trim().split(/\s+/).filter(term => term.length > 0);

  if (queryTerms.length === 0) {
    return text;
  }

  let highlightedText = text;
  queryTerms.forEach(term => {
    const escapedTerm = term.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
    const regex = new RegExp(`(${escapedTerm})`, 'gi');
    highlightedText = highlightedText.replace(regex, '<mark>$1</mark>');
  });

  return highlightedText;
};


/**
 * 执行搜索请求
 */
const performSearch = async () => {
  if (!query.value.trim()) {
    searchResults.value = [];
    spellingSuggestion.value = null; // 清空建议
    searched.value = false;
    return;
  }

  searched.value = true;
  spellingSuggestion.value = null; // 每次搜索前清空之前的建议

  const currentQuery = query.value; // 捕获当前查询值，防止在异步请求期间被修改

  // 1. 发送主搜索请求并处理
  axios.get<BackendResponse>(`http://localhost:8080/search`, {
    params: {
      query: currentQuery
    }
  })
  .then(response => {
    // 只有当查询值没有被用户更改时才更新UI，避免旧请求覆盖新请求
    if (query.value === currentQuery) {
      if (response.data.success && response.data.data) {
        searchResults.value = response.data.data.map(lecture => ({
          ...lecture,
          isExpanded: false
        }));
      } else {
        console.error('搜索失败:', response.data.errorMsg);
        searchResults.value = [];
      }
    }
  })
  .catch(error => {
    if (query.value === currentQuery) {
      console.error('搜索请求出错:', error);
      searchResults.value = [];
    }
  });

  // 2. 发送拼写建议请求并处理 (不阻塞主搜索)
  axios.get<SpellingAdviceResponse>(`http://localhost:8080/search/advice`, {
    params: {
      query: currentQuery
    }
  })
  .then(response => {
    // 只有当查询值没有被用户更改时才更新UI，避免旧请求覆盖新请求
    if (query.value === currentQuery) {
      if (response.data.success && response.data.data) {
        spellingSuggestion.value = response.data.data;
      } else {
        spellingSuggestion.value = null;
      }
    }
  })
  .catch(error => {
    if (query.value === currentQuery) {
      console.error('拼写建议请求出错:', error);
      spellingSuggestion.value = null;
    }
  });
};

/**
 * 切换讲座内容的展开/收起状态
 * @param lecture 当前点击的讲座对象
 */
const toggleLectureDetail = (lecture: LectureDocumentVO) => {
  if (lecture.isExpanded === undefined) {
    lecture.isExpanded = true;
  } else {
    lecture.isExpanded = !lecture.isExpanded;
  }
};

/**
 * 应用拼写建议，将建议词填充到搜索框并执行搜索
 * @param suggestion 拼写建议词
 */
const applySuggestion = (suggestion: string) => {
  query.value = suggestion;
  performSearch();
};
</script>

<style scoped>
/* ... (现有样式，保持不变直到 .search-header 部分) ... */

/* 新增高亮样式 */
.lecture-title :deep(mark),
.lecture-preview :deep(mark) {
  background-color: transparent; /* 移除默认的黄色背景 */
  text-decoration: underline wavy yellow; /* 黄色波浪线 */
  text-decoration-thickness: 2px; /* 波浪线粗细 */
  color: inherit; /* 保持文本颜色 */
  padding: 0; /* 移除可能的 padding */
}
/* 兼容性前缀，某些旧浏览器可能需要 */
.lecture-title :deep(mark),
.lecture-preview :deep(mark) {
  -webkit-text-decoration: underline wavy yellow;
  -webkit-text-decoration-thickness: 2px;
}


/* ------------------------------------------------------------- */
/* 以下为 SearchPartInterface.vue 的其余样式，保持不变 */
/* ------------------------------------------------------------- */
.search-container {
  display: flex;
  flex-direction: column;
  height: 100%; /* 占满父容器（right-panel）的高度 */
  width: 100%; /* 占满父容器的宽度 */
  background: #f8fafc;
  padding: 0;
  overflow: hidden; /* 防止内容溢出 */
}

.search-header {
  display: flex;
  justify-content: center; /* 标题居中 */
  align-items: center;
  padding: 1rem 1.5rem;
  background: linear-gradient(135deg, #4CAF50 0%, #8BC34A 100%); /* 不同的渐变色 */
  color: white;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  flex-shrink: 0;
}

.search-title {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 600;
}

.search-input-group {
  display: flex;
  padding: 1rem 1.5rem; /* 增加 padding */
  flex-shrink: 0;
}

.search-input {
  flex-grow: 1;
  padding: 10px 15px;
  border: 2px solid #e2e8f0;
  border-radius: 28px;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.2s ease;
}

.search-input:focus {
  border-color: #4CAF50;
}

.search-button {
  padding: 10px 20px;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 28px;
  cursor: pointer;
  font-size: 1rem;
  margin-left: 1rem;
  transition: all 0.2s ease;
  flex-shrink: 0;
}

.search-button:hover {
  background-color: #388E3C;
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(76, 175, 80, 0.4);
}

.search-button:disabled {
  background: #94a3b8;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* 新增：拼写建议样式 */
.spelling-suggestion-box {
  padding: 0.8rem 1.5rem;
  background-color: #fffde7; /* 淡黄色背景 */
  border-bottom: 1px solid #ffe082; /* 底部边框 */
  color: #616161;
  font-size: 0.95rem;
  display: flex;
  align-items: center;
  justify-content: center; /* 文本居中 */
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  margin: 0 1.5rem; /* 与搜索框对齐，但左右留白 */
  border-radius: 8px; /* 圆角 */
  margin-top: -0.5rem; /* 向上微调，更靠近搜索框 */
  animation: fadeInDown 0.3s ease-out; /* 淡入动画 */
}

@keyframes fadeInDown {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.spelling-suggestion-box p {
  margin: 0;
  display: flex;
  align-items: center;
}

.suggestion-text {
  color: #007bff;
  font-weight: bold;
  cursor: pointer;
  text-decoration: underline;
  margin-left: 0.5em;
  transition: color 0.2s ease;
}

.suggestion-text:hover {
  color: #0056b3;
}

.suggestion-tip {
  font-size: 0.85rem;
  color: #9e9e9e;
  margin-left: 0.5em;
}


.search-results {
  flex: 1;
  overflow-y: auto;
  padding: 0 1.5rem 1rem;
  scroll-behavior: smooth;
}

.search-results h2 {
  color: #333;
  margin-bottom: 15px;
  font-size: 1.25rem;
}

.result-list {
  list-style: none;
  padding: 0;
}

.result-item {
  margin-bottom: 20px; /* 增加列表项间距 */
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  overflow: hidden; /* 确保虚化效果在边界内 */
  transition: box-shadow 0.2s ease;
}

.result-item:hover {
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.lecture-link {
  display: block; /* 使整个区域可点击 */
  padding: 1rem 1.5rem;
  text-decoration: none;
  color: inherit; /* 继承父元素颜色 */
}

/* 注意：这里的 lecture-title 样式可能被 :deep(mark) 的样式覆盖 */
.lecture-title {
  color: #007bff; /* 保持链接颜色 */
  margin-top: 0;
  margin-bottom: 0.8rem;
  font-size: 1.1rem; /* 稍微放大标题 */
  font-weight: 600;
  cursor: pointer;
}

.lecture-preview {
  position: relative;
  overflow: hidden;
  line-height: 1.6;
  font-size: 0.95rem;
  color: #555;
  transition: max-height 0.4s ease-out; /* 高度过渡动画 */
}

/* 虚化效果 */
.lecture-preview::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 40px; /* 虚化区域的高度 */
  background: linear-gradient(to bottom, rgba(255, 255, 255, 0) 0%, rgba(255, 255, 255, 1) 100%);
  pointer-events: none; /* 确保不影响点击 */
  transition: opacity 0.4s ease-out;
}

/* 展开时虚化效果消失 */
.lecture-preview.expanded::after {
  opacity: 0;
}

.lecture-preview p {
  margin: 0; /* 移除 p 元素的默认外边距 */
  white-space: pre-wrap; /* 推荐使用这个，保留换行符并允许自动换行 */
  word-break: break-word; /* 确保长单词也能正常换行，防止溢出 */
}

.read-more-toggle {
  text-align: right;
  margin-top: 0.8rem;
  font-size: 0.9rem;
  color: #007bff;
  cursor: pointer;
  transition: color 0.2s ease;
}

.read-more-toggle:hover {
  color: #0056b3;
}

.no-results {
  margin-top: 40px; /* 增加顶部间距 */
  padding: 30px; /* 增加内边距 */
  background-color: #ffffff; /* 纯白背景 */
  border: 1px solid #e0e0e0; /* 更柔和的边框 */
  border-radius: 12px; /* 更大的圆角 */
  text-align: center;
  color: #616161; /* 更柔和的文本颜色 */
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.08); /* 增加阴影，使其更突出 */
  display: flex; /* 使用 flexbox 布局内容 */
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px; /* 最小高度 */
}

.no-results-icon {
  margin-bottom: 20px; /* 图标下方间距 */
  color: #9e9e9e; /* 图标颜色 */
}

.no-results-icon svg {
  width: 60px; /* 放大图标 */
  height: 60px;
}

.no-results-message {
  font-size: 1.2rem; /* 增大主提示文本 */
  font-weight: 600; /* 加粗 */
  color: #333; /* 更深的颜色 */
  margin-bottom: 10px;
}

.no-results-tip {
  font-size: 0.95rem; /* 提示文本大小 */
  color: #757575; /* 提示文本颜色 */
  line-height: 1.5;
}

/* 滚动条样式 */
.search-results::-webkit-scrollbar {
  width: 6px;
}

.search-results::-webkit-scrollbar-track {
  background: transparent;
}

.search-results::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 3px;
}

.search-results::-webkit-scrollbar-thumb:hover {
  background: #94a3b8;
}

/* 媒体查询：小屏幕下的布局调整 - 保持不变 */
@media (max-width: 1024px) {
  .search-container {
    height: auto;
    border-top: 1px solid #e2e8f0;
  }

  .search-header {
    padding: 0.75rem 1rem;
  }
  
  .search-title {
    font-size: 1.25rem;
  }

  .search-input-group {
    padding: 0.75rem 1rem;
  }

  .search-results {
    padding: 0 1rem 0.75rem;
  }
}
</style>