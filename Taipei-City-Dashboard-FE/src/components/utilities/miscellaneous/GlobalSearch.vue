<!-- Developed by Taipei Urban Intelligence Center 2023-2024-->

<script setup>
import { ref, computed, watch, onMounted, onUnmounted } from 'vue';
import { useRouter } from 'vue-router';
import { useContentStore } from '../../../store/contentStore';
import { useDialogStore } from '../../../store/dialogStore';
import http from '../../../router/axios';
import DashboardComponent from '../../../dashboardComponent/DashboardComponent.vue';

const router = useRouter();
const contentStore = useContentStore();
const dialogStore = useDialogStore();

const isSearchOpen = ref(false);
const searchQuery = ref('');
const searchResults = ref([]);
const isLoading = ref(false);
const selectedIndex = ref(-1);
const searchInput = ref(null);

// 搜尋結果分類
const categorizedResults = computed(() => {
  const results = {
    components: [],
    dashboards: []
  };
  
  searchResults.value.forEach(item => {
    if (item.type === 'component') {
      results.components.push(item);
    } else if (item.type === 'dashboard') {
      results.dashboards.push(item);
    }
  });
  
  return results;
});

// 執行搜尋
const performSearch = async () => {
  if (searchQuery.value.trim().length < 2) {
    searchResults.value = [];
    return;
  }
  
  isLoading.value = true;
  
  try {
    // 搜尋組件
    const componentResponse = await http.get('/component/', {
      params: {
        pagesize: 20,
        searchbyname: searchQuery.value,
        searchbyindex: searchQuery.value
      }
    });
    
    const components = componentResponse.data.data || [];
    
    // 搜尋儀表板
    const dashboardResponse = await http.get('/dashboard/');
    const dashboardData = dashboardResponse.data.data || {};
    
    let dashboards = [];
    Object.entries(dashboardData).forEach(([city, dashboardArray]) => {
      if (city !== 'personal' && Array.isArray(dashboardArray)) {
        dashboards = dashboards.concat(
          dashboardArray
            .filter(dashboard => 
              dashboard.name.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
              dashboard.index.toLowerCase().includes(searchQuery.value.toLowerCase())
            )
            .map(dashboard => ({
              ...dashboard,
              city,
              type: 'dashboard'
            }))
        );
      }
    });
    
    // 合併結果
    const results = [
      ...components.map(component => ({
        ...component,
        type: 'component'
      })),
      ...dashboards
    ];
    
    searchResults.value = results.slice(0, 10); // 限制結果數量
    selectedIndex.value = -1;
    
  } catch (error) {
    console.error('搜尋時發生錯誤：', error);
    dialogStore.showNotification('fail', '搜尋時發生錯誤');
  } finally {
    isLoading.value = false;
  }
};

// 防抖搜尋
let searchTimeout;
watch(searchQuery, () => {
  clearTimeout(searchTimeout);
  searchTimeout = setTimeout(performSearch, 300);
});

// 開啟搜尋
const openSearch = () => {
  isSearchOpen.value = true;
  setTimeout(() => {
    searchInput.value?.focus();
  }, 100);
};

// 關閉搜尋
const closeSearch = () => {
  isSearchOpen.value = false;
  searchQuery.value = '';
  searchResults.value = [];
  selectedIndex.value = -1;
};

// 選擇搜尋結果
const selectResult = (result) => {
  if (result.type === 'component') {
    // 尋找包含此組件的儀表板
    navigateToComponentDashboard(result);
  } else if (result.type === 'dashboard') {
    // 直接跳轉到儀表板
    navigateToDashboard(result);
  }
  closeSearch();
};

// 跳轉到包含組件的儀表板
const navigateToComponentDashboard = async (component) => {
  try {
    // 搜尋包含此組件的儀表板
    const dashboardResponse = await http.get('/dashboard/');
    const dashboardData = dashboardResponse.data.data || {};
    
    let targetDashboard = null;
    let targetCity = null;
    
    // 遍歷所有儀表板尋找包含此組件的儀表板
    for (const [city, dashboards] of Object.entries(dashboardData)) {
      if (city === 'personal') continue;
      
      for (const dashboard of dashboards) {
        if (dashboard.components && dashboard.components.includes(component.id)) {
          targetDashboard = dashboard;
          targetCity = city;
          break;
        }
      }
      
      if (targetDashboard) break;
    }
    
    if (targetDashboard) {
      // 跳轉到包含此組件的儀表板
      router.push({
        name: 'dashboard',
        query: {
          index: targetDashboard.index,
          city: targetCity
        }
      });
      
      dialogStore.showNotification('success', `已跳轉到包含「${component.name}」的儀表板`);
    } else {
      // 如果沒有找到包含此組件的儀表板，跳轉到組件詳情頁
      router.push({
        name: 'component-info',
        params: { index: component.index }
      });
      
      dialogStore.showNotification('info', `未找到包含此組件的儀表板，已跳轉到組件詳情頁`);
    }
  } catch (error) {
    console.error('跳轉時發生錯誤：', error);
    dialogStore.showNotification('fail', '跳轉時發生錯誤');
  }
};

// 跳轉到儀表板
const navigateToDashboard = (dashboard) => {
  router.push({
    name: 'dashboard',
    query: {
      index: dashboard.index,
      city: dashboard.city
    }
  });
  
  dialogStore.showNotification('success', `已跳轉到「${dashboard.name}」儀表板`);
};

// 鍵盤事件處理
const handleKeydown = (event) => {
  if (!isSearchOpen.value) return;
  
  const totalResults = searchResults.value.length;
  
  switch (event.key) {
    case 'ArrowDown':
      event.preventDefault();
      selectedIndex.value = (selectedIndex.value + 1) % totalResults;
      break;
      
    case 'ArrowUp':
      event.preventDefault();
      selectedIndex.value = selectedIndex.value <= 0 ? totalResults - 1 : selectedIndex.value - 1;
      break;
      
    case 'Enter':
      event.preventDefault();
      if (selectedIndex.value >= 0 && searchResults.value[selectedIndex.value]) {
        selectResult(searchResults.value[selectedIndex.value]);
      }
      break;
      
    case 'Escape':
      event.preventDefault();
      closeSearch();
      break;
  }
};

// 快捷鍵處理
const handleGlobalKeydown = (event) => {
  // Ctrl/Cmd + K 開啟搜尋
  if ((event.ctrlKey || event.metaKey) && event.key === 'k') {
    event.preventDefault();
    openSearch();
  }
};

onMounted(() => {
  document.addEventListener('keydown', handleGlobalKeydown);
});

onUnmounted(() => {
  document.removeEventListener('keydown', handleGlobalKeydown);
});
</script>

<template>
  <!-- 搜尋按鈕 -->
  <button 
    class="search-trigger"
    @click="openSearch"
    title="搜尋 (Ctrl+K)"
  >
    <span>search</span>
  </button>

  <!-- 搜尋對話框 -->
  <Teleport to="body">
    <Transition name="search-dialog">
      <div 
        v-if="isSearchOpen" 
        class="search-overlay"
        @click="closeSearch"
      >
        <div 
          class="search-dialog"
          @click.stop
        >
          <!-- 搜尋輸入框 -->
          <div class="search-input-container">
            <span class="search-icon">search</span>
            <input
              ref="searchInput"
              v-model="searchQuery"
              type="text"
              placeholder="搜尋組件或儀表板..."
              class="search-input"
              @keydown="handleKeydown"
            />
            <button 
              class="search-close"
              @click="closeSearch"
            >
              <span>close</span>
            </button>
          </div>

          <!-- 搜尋結果 -->
          <div class="search-results">
            <!-- 載入中 -->
            <div v-if="isLoading" class="search-loading">
              <div class="loading-spinner"></div>
              <p>搜尋中...</p>
            </div>

            <!-- 無結果 -->
            <div v-else-if="searchQuery && searchResults.length === 0" class="search-no-results">
              <span>search_off</span>
              <p>找不到相關結果</p>
            </div>

            <!-- 搜尋結果列表 -->
            <div v-else-if="searchResults.length > 0" class="search-results-list">
              <!-- 組件結果 -->
              <div v-if="categorizedResults.components.length > 0" class="search-category">
                <h3>組件</h3>
                <div
                  v-for="(component, index) in categorizedResults.components"
                  :key="`component-${component.id}`"
                  :class="{
                    'search-result-item': true,
                    'selected': selectedIndex === searchResults.findIndex(r => r.id === component.id && r.type === 'component')
                  }"
                  @click="selectResult(component)"
                >
                  <div class="result-icon">
                    <span>dashboard</span>
                  </div>
                  <div class="result-content">
                    <h4>{{ component.name }}</h4>
                    <p>{{ component.index }} | {{ component.city }}</p>
                    <span class="result-type">組件</span>
                  </div>
                </div>
              </div>

              <!-- 儀表板結果 -->
              <div v-if="categorizedResults.dashboards.length > 0" class="search-category">
                <h3>儀表板</h3>
                <div
                  v-for="(dashboard, index) in categorizedResults.dashboards"
                  :key="`dashboard-${dashboard.index}`"
                  :class="{
                    'search-result-item': true,
                    'selected': selectedIndex === searchResults.findIndex(r => r.index === dashboard.index && r.type === 'dashboard')
                  }"
                  @click="selectResult(dashboard)"
                >
                  <div class="result-icon">
                    <span>{{ dashboard.icon }}</span>
                  </div>
                  <div class="result-content">
                    <h4>{{ dashboard.name }}</h4>
                    <p>{{ dashboard.index }} | {{ dashboard.city }}</p>
                    <span class="result-type">儀表板</span>
                  </div>
                </div>
              </div>
            </div>

            <!-- 搜尋提示 -->
            <div v-else class="search-hint">
              <p>輸入關鍵字搜尋組件或儀表板</p>
              <div class="search-shortcuts">
                <span><kbd>↑</kbd><kbd>↓</kbd> 導航</span>
                <span><kbd>Enter</kbd> 選擇</span>
                <span><kbd>Esc</kbd> 關閉</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped lang="scss">
.search-trigger {
  display: flex;
  align-items: center;
  padding: 0.5rem;
  border: none;
  border-radius: 0.5rem;
  background-color: var(--color-component-background);
  color: var(--color-complement-text);
  font-size: var(--font-m);
  cursor: pointer;
  transition: all 0.2s ease;
  z-index: 10;

  &:hover {
    background-color: var(--color-border);
    color: var(--color-normal-text);
  }

  span {
    font-family: var(--font-icon);
  }
}

.search-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: flex-start;
  justify-content: center;
  padding-top: 10vh;
  z-index: 1000;
}

.search-dialog {
  width: 90%;
  max-width: 600px;
  background-color: var(--color-component-background);
  border-radius: 1rem;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
  overflow: hidden;
}

.search-input-container {
  display: flex;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid var(--color-border);

  .search-icon {
    font-family: var(--font-icon);
    color: var(--color-complement-text);
    margin-right: 0.75rem;
  }

  .search-input {
    flex: 1;
    border: none;
    background: transparent;
    color: var(--color-normal-text);
    font-size: var(--font-m);
    outline: none;

    &::placeholder {
      color: var(--color-complement-text);
    }
  }

  .search-close {
    background: none;
    border: none;
    color: var(--color-complement-text);
    cursor: pointer;
    padding: 0.25rem;
    border-radius: 0.25rem;
    transition: background-color 0.2s ease;

    &:hover {
      background-color: var(--color-border);
    }

    span {
      font-family: var(--font-icon);
    }
  }
}

.search-results {
  max-height: 400px;
  overflow-y: auto;
  padding: 1rem 0;
}

.search-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
  color: var(--color-complement-text);

  .loading-spinner {
    width: 2rem;
    height: 2rem;
    border: 3px solid var(--color-border);
    border-top: 3px solid var(--color-highlight);
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin-bottom: 1rem;
  }
}

.search-no-results {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
  color: var(--color-complement-text);

  span {
    font-family: var(--font-icon);
    font-size: 3rem;
    margin-bottom: 1rem;
  }
}

.search-category {
  margin-bottom: 1rem;

  h3 {
    padding: 0 1rem 0.5rem;
    font-size: var(--font-s);
    color: var(--color-complement-text);
    text-transform: uppercase;
    letter-spacing: 0.05em;
    border-bottom: 1px solid var(--color-border);
    margin-bottom: 0.5rem;
  }
}

.search-result-item {
  display: flex;
  align-items: center;
  padding: 0.75rem 1rem;
  cursor: pointer;
  transition: background-color 0.2s ease;

  &:hover,
  &.selected {
    background-color: var(--color-border);
  }

  .result-icon {
    width: 2.5rem;
    height: 2.5rem;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: var(--color-highlight);
    border-radius: 0.5rem;
    margin-right: 1rem;

    span {
      font-family: var(--font-icon);
      color: white;
    }
  }

  .result-content {
    flex: 1;
    position: relative;

    h4 {
      font-size: var(--font-m);
      color: var(--color-normal-text);
      margin-bottom: 0.25rem;
    }

    p {
      font-size: var(--font-s);
      color: var(--color-complement-text);
      margin: 0;
    }

    .result-type {
      position: absolute;
      top: 0;
      right: 0;
      background-color: var(--color-highlight);
      color: white;
      padding: 0.25rem 0.5rem;
      border-radius: 0.25rem;
      font-size: var(--font-xs);
    }
  }
}

.search-hint {
  text-align: center;
  padding: 2rem;
  color: var(--color-complement-text);

  p {
    margin-bottom: 1rem;
  }

  .search-shortcuts {
    display: flex;
    justify-content: center;
    gap: 1rem;
    font-size: var(--font-s);

    kbd {
      background-color: var(--color-border);
      padding: 0.25rem 0.5rem;
      border-radius: 0.25rem;
      font-family: monospace;
    }
  }
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

// 動畫效果
.search-dialog-enter-active,
.search-dialog-leave-active {
  transition: all 0.3s ease;
}

.search-dialog-enter-from {
  opacity: 0;
  transform: translateY(-50px) scale(0.9);
}

.search-dialog-leave-to {
  opacity: 0;
  transform: translateY(-50px) scale(0.9);
}
</style>