<template>
  <div class="search-container">
    <header class="search-header">
      <h1 class="search-title">讲座检索</h1>
    </header>

    <div class="search-input-group">
      <CustomSelect
        v-model="searchType"
        :options="searchTypeOptions"
        class="custom-select-wrapper-in-group"
      />
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
import CustomSelect from './CustomSelect.vue'; // 导入 CustomSelect 组件

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

interface SpellingAdviceResponse {
  success: boolean;
  errorMsg: string | null;
  data: string | null;
}

const query = ref<string>('');
const searchType = ref<number>(0); // 搜索类型，默认为0 (全文搜索)

// 定义 CustomSelect 的选项数据
const searchTypeOptions = [
  { value: 0, label: '全文搜索' },
  { value: 1, label: '标题搜索' },
  { value: 2, label: '讲师搜索' },
];

const searchResults = ref<LectureDocumentVO[]>([]);
const searched = ref<boolean>(false);
const spellingSuggestion = ref<string | null>(null);

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
    spellingSuggestion.value = null;
    searched.value = false;
    return;
  }

  searched.value = true;
  spellingSuggestion.value = null;

  const currentQuery = query.value;
  const currentSearchType = searchType.value; // 捕获当前搜索类型

  // 1. 发送主搜索请求并立即处理其结果
  axios.get<BackendResponse>(`http://localhost:8080/search`, {
    params: {
      query: currentQuery,
      type: currentSearchType // 传递 type 参数
    }
  })
  .then(response => {
    // 检查当前 query.value 和 searchType.value 是否与发起请求时相同
    if (query.value === currentQuery && searchType.value === currentSearchType) {
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
    if (query.value === currentQuery && searchType.value === currentSearchType) {
      console.error('搜索请求出错:', error);
      searchResults.value = [];
    }
  });

  // 2. 发送拼写建议请求并立即处理其结果 (不阻塞主搜索)
  // 拼写建议通常只针对查询字符串，不依赖搜索类型
  axios.get<SpellingAdviceResponse>(`http://localhost:8080/search/advice`, {
    params: {
      query: currentQuery
    }
  })
  .then(response => {
    if (query.value === currentQuery) { // 拼写建议只需检查 query 是否一致
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
  height: 100%;
  width: 100%;
  background: #f8fafc;
  padding: 0;
  overflow: hidden;
}

.search-header {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 1rem 1.5rem;
  background: linear-gradient(135deg, #4CAF50 0%, #8BC34A 100%);
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
  padding: 1rem 1.5rem;
  flex-shrink: 0;
  align-items: center; /* 垂直居中对齐 */
  gap: 1rem; /* 组件之间的间距 */
}

/* 为 CustomSelect 在 SearchInputGroup 中的布局添加样式 */
/* 注意：这里的样式是针对 CustomSelect 的外层容器 .custom-select-wrapper */
.custom-select-wrapper-in-group {
  /* 可以选择不设置特定宽度，让其内容决定宽度 */
  /* 或者设置一个最小宽度 / 最大宽度 */
  min-width: 120px; /* 示例：给定一个最小宽度 */
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

/* 拼写建议样式 */
.spelling-suggestion-box {
  padding: 0.8rem 1.5rem;
  background-color: #fffde7;
  border-bottom: 1px solid #ffe082;
  color: #616161;
  font-size: 0.95rem;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  margin: 0 1.5rem;
  border-radius: 8px;
  margin-top: 0.5rem; /* 调整间距 */
  animation: fadeInDown 0.3s ease-out;
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
  margin-bottom: 20px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.05);
  overflow: hidden;
  transition: box-shadow 0.2s ease;
}

.result-item:hover {
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.lecture-link {
  display: block;
  padding: 1rem 1.5rem;
  text-decoration: none;
  color: inherit;
}

.lecture-title {
  color: #007bff;
  margin-top: 0;
  margin-bottom: 0.8rem;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
}

.lecture-preview {
  position: relative;
  overflow: hidden;
  line-height: 1.6;
  font-size: 0.95rem;
  color: #555;
  transition: max-height 0.4s ease-out;
}

.lecture-preview::after {
  content: '';
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 40px;
  background: linear-gradient(to bottom, rgba(255, 255, 255, 0) 0%, rgba(255, 255, 255, 1) 100%);
  pointer-events: none;
  transition: opacity 0.4s ease-out;
}

.lecture-preview.expanded::after {
  opacity: 0;
}

.lecture-preview p {
  margin: 0;
  white-space: pre-wrap;
  word-break: break-word;
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
  margin-top: 40px;
  padding: 30px;
  background-color: #ffffff;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  text-align: center;
  color: #616161;
  box-shadow: 0 5px 15px rgba(0, 0, 0, 0.08);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
}

.no-results-icon {
  margin-bottom: 20px;
  color: #9e9e9e;
}

.no-results-icon svg {
  width: 60px;
  height: 60px;
}

.no-results-message {
  font-size: 1.2rem;
  font-weight: 600;
  color: #333;
  margin-bottom: 10px;
}

.no-results-tip {
  font-size: 0.95rem;
  color: #757575;
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

/* 媒体查询：小屏幕下的布局调整 */
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
    flex-direction: column; /* 小屏幕下堆叠 */
    align-items: stretch; /* 填充宽度 */
  }

  /* CustomSelect 在小屏幕下也应占据全宽 */
  .custom-select-wrapper-in-group {
    width: 100%;
    margin-bottom: 0.5rem; /* 下拉框下方间距 */
  }

  .search-input {
    width: 100%; /* 小屏幕下宽度占满 */
    margin-bottom: 0.5rem; /* 输入框下方间距 */
  }

  .search-button {
    width: 100%; /* 小屏幕下宽度占满 */
    margin-left: 0; /* 移除左侧间距 */
  }

  .search-results {
    padding: 0 1rem 0.75rem;
  }
}
</style>