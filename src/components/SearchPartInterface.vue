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

    <div v-if="searchResults.length > 0" class="search-results">
      <h2>搜索结果：</h2>
      <ul class="result-list">
        <li v-for="lecture in searchResults" :key="lecture.id" class="result-item">
          <a href="#" @click.prevent="viewLectureDetail(lecture.id)" class="lecture-title">
            {{ lecture.title }}
          </a>
        </li>
      </ul>
    </div>

    <div v-else-if="searched && searchResults.length === 0" class="no-results">
      <p>没有找到相关讲座。</p>
    </div>

    <div v-if="showDetailModal" class="modal-overlay" @click.self="closeDetailModal">
      <div class="modal-content">
        <button class="close-button" @click="closeDetailModal">X</button>
        <h3>{{ currentLecture.title }}</h3>
        <div class="lecture-content">
          <p>{{ currentLecture.content }}</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import axios from 'axios';

// 定义讲座文档的接口
interface LectureDocumentVO {
  id: number;
  title: string;
  content?: string;
}

// 定义后端响应的接口
interface BackendResponse {
  success: boolean;
  errorMsg: string | null;
  data: LectureDocumentVO[] | LectureDocumentVO | null; // 修正 data 类型以同时支持数组和对象
  total: number | null;
}

const query = ref<string>('');
const searchResults = ref<LectureDocumentVO[]>([]);
const searched = ref<boolean>(false);

const showDetailModal = ref<boolean>(false);
const currentLecture = ref<LectureDocumentVO>({ id: 0, title: '', content: '' });

/**
 * 执行搜索请求
 */
const performSearch = async () => {
  if (!query.value.trim()) {
    searchResults.value = [];
    searched.value = false;
    return;
  }

  searched.value = true;
  try {
    const response = await axios.get<BackendResponse>(`http://localhost:8080/search`, {
      params: {
        query: query.value
      }
    });

    if (response.data.success && response.data.data) {
      // 确保 data 是数组类型 (对于 /search 接口)
      if (Array.isArray(response.data.data)) {
        searchResults.value = response.data.data;
      } else {
        // 如果后端在搜索结果中返回单个对象而不是数组，这可能是个警告，但也兼容处理
        console.warn("Backend search response 'data' is not an array for /search. Please check backend API.");
        searchResults.value = [response.data.data as LectureDocumentVO];
      }
    } else {
      console.error('搜索失败:', response.data.errorMsg);
      searchResults.value = [];
    }
  } catch (error) {
    console.error('搜索请求出错:', error);
    searchResults.value = [];
  }
};

/**
 * 查看讲座详情
 * @param id 讲座ID
 */
const viewLectureDetail = async (id: number) => {
  try {
    const response = await axios.get<BackendResponse>(`http://localhost:8080/search/${id}`);
    if (response.data.success && response.data.data) {
      // **** 关键修改在这里 ****
      // 后端返回的 data 应该是一个 LectureDocumentVO 对象，直接赋值
      currentLecture.value = response.data.data as LectureDocumentVO;
      showDetailModal.value = true;
    } else {
      console.error('获取讲座详情失败:', response.data.errorMsg);
      alert('无法获取讲座详情。');
    }
  } catch (error) {
    console.error('获取讲座详情请求出错:', error);
    alert('请求讲座详情时发生错误。');
  }
};

/**
 * 关闭文章详情弹窗
 */
const closeDetailModal = () => {
  showDetailModal.value = false;
  currentLecture.value = { id: 0, title: '', content: '' };
};
</script>

<style scoped>
/* 调整 SearchPartInterface 的样式以适应 Flex 布局 */
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
  margin-bottom: 10px;
  padding: 8px 0;
  border-bottom: 1px dashed #eee;
}

.result-item:last-child {
  border-bottom: none;
}

.lecture-title {
  color: #4f46e5;
  text-decoration: none;
  font-size: 1rem;
  font-weight: normal;
  display: block;
  padding: 5px 0;
  transition: color 0.2s ease;
}

.lecture-title:hover {
  text-decoration: underline;
  color: #7c3aed;
}

.no-results {
  margin-top: 20px;
  padding: 15px;
  background-color: #f8f8f8;
  border: 1px solid #ddd;
  border-radius: 5px;
  text-align: center;
  color: #666;
}

/* 弹窗样式保持不变 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  padding: 30px;
  border-radius: 8px;
  width: 90%;
  max-width: 700px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
  position: relative;
  max-height: 80vh;
  overflow-y: auto;
}

.modal-content h3 {
  margin-top: 0;
  color: #333;
  font-size: 24px;
  margin-bottom: 15px;
}

.lecture-content {
  white-space: pre-wrap;
  line-height: 1.6;
  color: #555;
  font-size: 15px;
}

.close-button {
  position: absolute;
  top: 15px;
  right: 15px;
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #888;
}

.close-button:hover {
  color: #333;
}

/* 响应式设计 */
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
</style>