<template>
  <div class="custom-select-wrapper" ref="selectWrapperRef">
    <div
      class="custom-select-trigger"
      :class="{ 'is-open': isOpen, 'focused': isFocused }"
      @click="toggleDropdown"
      @focus="isFocused = true"
      @blur="isFocused = false"
      tabindex="0"
    >
      <span class="selected-value">{{ selectedOptionLabel }}</span>
      <span class="dropdown-arrow" :class="{ 'rotate': isOpen }">
        <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <polyline points="6 9 12 15 18 9"></polyline>
        </svg>
      </span>
    </div>

    <Transition name="dropdown-fade-scale">
      <div v-if="isOpen" class="custom-select-options">
        <div
          v-for="option in options"
          :key="option.value"
          class="option-item"
          :class="{ 'is-active': option.value === modelValue }"
          @click="selectOption(option.value)"
        >
          {{ option.label }}
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue';

interface SelectOption {
  value: number;
  label: string;
}

const props = defineProps<{
  modelValue: number; // 当前选中的值
  options: SelectOption[]; // 选项数组 [{ value: 0, label: '全文搜索' }, ...]
}>();

const emit = defineEmits(['update:modelValue']);

const isOpen = ref(false); // 控制下拉列表的显示/隐藏
const isFocused = ref(false); // 控制聚焦状态
const selectWrapperRef = ref<HTMLElement | null>(null); // 引用最外层 div

// 计算当前选中项的标签
const selectedOptionLabel = computed(() => {
  const selected = props.options.find(option => option.value === props.modelValue);
  return selected ? selected.label : '请选择'; // 默认值
});

// 切换下拉列表显示状态
const toggleDropdown = () => {
  isOpen.value = !isOpen.value;
};

// 选择一个选项
const selectOption = (value: number) => {
  emit('update:modelValue', value); // 更新父组件的 modelValue
  isOpen.value = false; // 关闭下拉列表
};

// 点击外部关闭下拉列表
const handleClickOutside = (event: MouseEvent) => {
  if (selectWrapperRef.value && !selectWrapperRef.value.contains(event.target as Node)) {
    isOpen.value = false;
  }
};

onMounted(() => {
  document.addEventListener('click', handleClickOutside);
});

onUnmounted(() => {
  document.removeEventListener('click', handleClickOutside);
});
</script>

<style scoped>
.custom-select-wrapper {
  position: relative;
  display: inline-block; /* 或 flex-shrink: 0; 保持其不被压缩 */
  user-select: none;
  flex-shrink: 0; /* 确保在 flex 容器中不被压缩 */
}

.custom-select-trigger {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 15px; /* 基础内边距 */
  padding-right: 20px; /* 为箭头留出空间 */
  border: 2px solid #e2e8f0;
  border-radius: 28px; /* 圆角 */
  background-color: white; /* 白色背景 */
  color: #374151; /* 文本颜色 */
  cursor: pointer;
  font-size: 1rem;
  outline: none; /* 移除默认的聚焦轮廓 */
  position: relative;
  z-index: 2; /* 确保在选项之上 */

  /* 绿白色调和动画 */
  border-color: #4CAF50; /* 绿色边框 */
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05); /* 轻微阴影 */
  transition: all 0.3s ease-in-out;
}

.custom-select-trigger:hover {
  border-color: #388E3C; /* 悬停时深绿色边框 */
  box-shadow: 0 4px 12px rgba(76, 175, 80, 0.2); /* 悬停时更明显的阴影 */
  transform: translateY(-2px); /* 轻微上浮 */
}

.custom-select-trigger.focused {
  border-color: #388E3C; /* 聚焦时深绿色边框 */
  box-shadow: 0 0 0 4px rgba(76, 175, 80, 0.4); /* 聚焦时绿色光晕，更大更明显 */
  transform: translateY(0); /* 聚焦时回到原位 */
}

.custom-select-trigger.is-open {
  border-color: #388E3C; /* 展开时保持绿色边框 */
  border-bottom-left-radius: 0; /* 展开时底部圆角取消 */
  border-bottom-right-radius: 0;
  box-shadow: 0 4px 12px rgba(76, 175, 80, 0.4); /* 展开时保持阴影 */
  transform: translateY(0); /* 展开时回到原位 */
}

.selected-value {
  flex-grow: 1;
}

.dropdown-arrow {
  width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: 10px;
  color: #4CAF50; /* 箭头颜色 */
  transition: transform 0.3s ease-in-out;
}

.dropdown-arrow.rotate {
  transform: rotate(180deg);
}

.custom-select-options {
  position: absolute;
  top: 100%; /* 定位到触发器下方 */
  left: 0;
  right: 0;
  background-color: white;
  border: 2px solid #4CAF50; /* 绿色边框 */
  border-top: none; /* 顶部无边框，与触发器连接 */
  border-bottom-left-radius: 8px; /* 底部圆角 */
  border-bottom-right-radius: 8px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1); /* 较明显的阴影 */
  max-height: 200px; /* 最大高度，超出滚动 */
  overflow-y: auto;
  z-index: 1; /* 确保在触发器下方 */
  transform-origin: top center; /* 动画原点 */
}

.option-item {
  padding: 10px 15px;
  cursor: pointer;
  color: #374151;
  transition: all 0.2s ease-in-out;
  border-bottom: 1px solid #f0f2f5; /* 选项之间的分隔线 */
}

.option-item:last-child {
  border-bottom: none; /* 最后一个选项没有底边框 */
}

.option-item:hover {
  background-color: #f0fdf4; /* 悬停时的淡绿色背景 */
  color: #388E3C; /* 悬停时的绿色文本 */
}

.option-item.is-active {
  background-color: #e6f6ed; /* 选中项的淡绿色背景 */
  color: #4CAF50; /* 选中项的绿色文本 */
  font-weight: 600;
}

/* 过渡动画 */
.dropdown-fade-scale-enter-active,
.dropdown-fade-scale-leave-active {
  transition: all 0.3s ease-in-out;
}

.dropdown-fade-scale-enter-from,
.dropdown-fade-scale-leave-to {
  opacity: 0;
  transform: scaleY(0.8); /* 从缩小状态进入/退出 */
}
</style>