<!-- Developed by Taipei Urban Intelligence Center 2023-2024-->

<script setup>
import { computed, onMounted, ref, watch } from "vue";
import http from "../../router/axios";
import DashboardComponent from "../../dashboardComponent/DashboardComponent.vue";

import { useDialogStore } from "../../store/dialogStore";
import { useContentStore } from "../../store/contentStore";

import DialogContainer from "./DialogContainer.vue";

const dialogStore = useDialogStore();
const contentStore = useContentStore();

const allComponents = ref(null);
const componentsSelected = ref([]);
const searchName = ref("");
const searchIndex = ref("");
const searchCity = ref("");
const isSearching = ref(false);

// 實時搜尋過濾
const availableComponents = computed(() => {
	const taken = contentStore.editDashboard.components?.map((item) => item.id) || [];
	let available = allComponents.value?.filter(
		(item) => !taken.includes(+item.id)
	) || [];

	// 客戶端實時過濾
	if (searchName.value || searchIndex.value || searchCity.value) {
		available = available.filter(item => {
			const nameMatch = !searchName.value || 
				item.name.toLowerCase().includes(searchName.value.toLowerCase());
			const indexMatch = !searchIndex.value || 
				item.index.toLowerCase().includes(searchIndex.value.toLowerCase());
			const cityMatch = !searchCity.value || 
				item.city.toLowerCase().includes(searchCity.value.toLowerCase());
			
			return nameMatch && indexMatch && cityMatch;
		});
	}

	return available;
});

// 防抖搜尋
let searchTimeout = null;
const debouncedSearch = () => {
	clearTimeout(searchTimeout);
	searchTimeout = setTimeout(handleSearch, 300);
};

// 監聽搜尋條件變化，實現實時搜尋
watch([searchName, searchIndex, searchCity], () => {
	if (allComponents.value) {
		// 如果已有資料，使用客戶端過濾
		return;
	}
	// 如果沒有資料，進行伺服器搜尋
	debouncedSearch();
});

async function handleSearch() {
	if (isSearching.value) return;
	
	isSearching.value = true;
	contentStore.loading = true;
	
	try {
		const response = await http.get(`/component/`, {
			params: {
				pagesize: 500, // 增加搜尋結果數量
				searchbyindex: searchIndex.value,
				searchbyname: searchName.value,
				city: searchCity.value,
			},
		});
		
		const data = response.data.data || [];
		const uniqueData = [...new Map(data
			.sort((a) => a.city === 'metrotaipei' ? 1 : -1)
			.map(item => [item.id, item]))
			.values()
		];
		
		allComponents.value = uniqueData;
	} catch (error) {
		console.error('搜尋組件時發生錯誤：', error);
		allComponents.value = [];
		dialogStore.showNotification("fail", "搜尋組件時發生錯誤");
	} finally {
		contentStore.loading = false;
		isSearching.value = false;
	}
}

// 清除所有搜尋條件
function clearAllSearch() {
	searchName.value = "";
	searchIndex.value = "";
	searchCity.value = "";
	handleSearch();
}

// 清除單一搜尋條件
function clearSearch(type) {
	switch (type) {
		case 'name':
			searchName.value = "";
			break;
		case 'index':
			searchIndex.value = "";
			break;
		case 'city':
			searchCity.value = "";
			break;
	}
}

function handleSubmit() {
	contentStore.editDashboard.components =
		contentStore.editDashboard.components?.concat(componentsSelected.value) ?? componentsSelected.value;
	handleClose();
}

function handleClose() {
	searchName.value = "";
	searchIndex.value = "";
	searchCity.value = "";
	componentsSelected.value = [];
	dialogStore.dialogs.addComponent = false;
	handleSearch();
}

onMounted(() => {
	handleSearch();
});
</script>

<template>
  <DialogContainer
    dialog="addComponent"
    @on-close="handleClose"
  >
    <div class="addcomponent">
      <div class="addcomponent-header">
        <h2>新增組件至儀表板</h2>
        <div class="addcomponent-header-search">
          <div class="addcomponent-header-search-inputs">
            <!-- 名稱搜尋 -->
            <div class="search-input-container">
              <input
                v-model="searchName"
                type="text"
                placeholder="以名稱搜尋"
                @input="debouncedSearch"
                @keypress.enter="handleSearch"
              >
              <span
                v-if="searchName"
                class="clear-btn"
                @click="clearSearch('name')"
              >cancel</span>
            </div>
            
            <!-- Index搜尋 -->
            <div class="search-input-container">
              <input
                v-model="searchIndex"
                type="text"
                placeholder="以Index搜尋"
                @input="debouncedSearch"
                @keypress.enter="handleSearch"
              >
              <span
                v-if="searchIndex"
                class="clear-btn"
                @click="clearSearch('index')"
              >cancel</span>
            </div>
            
            <!-- 城市搜尋 -->
            <div class="search-input-container">
              <input
                v-model="searchCity"
                type="text"
                placeholder="以城市搜尋"
                @input="debouncedSearch"
                @keypress.enter="handleSearch"
              >
              <span
                v-if="searchCity"
                class="clear-btn"
                @click="clearSearch('city')"
              >cancel</span>
            </div>
          </div>
          
          <div class="addcomponent-header-search-actions">
            <button 
              v-if="searchName || searchIndex || searchCity"
              @click="clearAllSearch"
              class="clear-all-btn"
            >
              清除全部
            </button>
            <button @click="handleClose">
              取消
            </button>
            <button
              v-if="componentsSelected?.length > 0"
              @click="handleSubmit"
              class="confirm-btn"
            >
              <span>add_chart</span>確認新增
            </button>
          </div>
        </div>
      </div>
      
      <!-- 搜尋狀態顯示 -->
      <div class="addcomponent-status">
        <p v-if="contentStore.loading || isSearching">
          <span class="loading-icon">hourglass_empty</span>
          搜尋中...
        </p>
        <p v-else-if="availableComponents?.length === 0 && allComponents">
          <span>search_off</span>
          找不到符合條件的組件
        </p>
        <p v-else>
          計 {{ availableComponents?.length || 0 }} 個組件符合篩選條件 | 共選取
          {{ componentsSelected?.length }} 個
        </p>
      </div>

      <!-- 組件列表 -->
      <div class="addcomponent-list">
        <div
          v-for="item in availableComponents"
          :key="`${item.id}-${item.city}`"
        >
          <input
            :id="`${item.name}-${item.city}`"
            v-model="componentsSelected"
            type="checkbox"
            :value="{ id: item.id, name: item.name, city: item.city }"
          >
          <label :for="`${item.name}-${item.city}`">
            <div class="addcomponent-list-item">
              <DashboardComponent
                :config="item"
                :city-tag="contentStore.cityManager.getTagList(item.city)"
                mode="preview"
              />
            </div>
          </label>
        </div>
      </div>
    </div>
  </DialogContainer>
</template>

<style scoped lang="scss">
.addcomponent {
	width: 700px;
	height: 600px;
	padding: 10px;

	&-header {
		h2 {
			font-size: var(--font-m);
		}

		&-search {
			display: flex;
			flex-direction: column;
			gap: var(--font-ms);
			margin-top: var(--font-ms);

			&-inputs {
				display: flex;
				gap: 0.5rem;
				flex-wrap: wrap;
			}

			&-actions {
				display: flex;
				justify-content: flex-end;
				gap: 0.5rem;
			}

			.search-input-container {
				position: relative;
				flex: 1;
				min-width: 150px;

				input {
					width: 100%;
					padding-right: 30px;
				}

				.clear-btn {
					position: absolute;
					right: 0.5rem;
					top: 0.4rem;
					margin-right: 4px;
					color: var(--color-complement-text);
					font-family: var(--font-icon);
					font-size: var(--font-m);
					cursor: pointer;
					transition: color 0.2s;

					&:hover {
						color: var(--color-highlight);
					}
				}
			}

			.clear-all-btn {
				padding: 4px 8px;
				border-radius: 5px;
				background-color: var(--color-border);
				color: var(--color-text);
				font-size: var(--font-ms);
				transition: background-color 0.2s;

				&:hover {
					background-color: var(--color-complement-text);
				}
			}

			.confirm-btn {
				display: flex;
				align-items: center;
				padding: 4px 8px;
				border-radius: 5px;
				background-color: var(--color-highlight);
				font-size: var(--font-ms);

				span {
					margin-right: 4px;
					font-family: var(--font-icon);
					font-size: calc(var(--font-ms) * var(--font-to-icon));
				}
			}

			button {
				padding: 4px 8px;
				border-radius: 5px;
				font-size: var(--font-ms);
				transition: opacity 0.2s;
				cursor: pointer;

				&:hover {
					opacity: 0.8;
				}
			}
		}
	}

	&-status {
		margin: 1rem 0 0.5rem;
		
		p {
			display: flex;
			align-items: center;
			gap: 0.5rem;
			font-size: var(--font-ms);
			color: var(--color-complement-text);

			span {
				font-family: var(--font-icon);
				font-size: var(--font-m);
			}

			.loading-icon {
				animation: spin 1s linear infinite;
			}
		}
	}

	&-list {
		max-height: calc(100% - 9rem);
		display: grid;
		grid-template-columns: 1fr 1fr;
		row-gap: var(--font-ms);
		column-gap: var(--font-ms);
		overflow-y: scroll;

		&-item {
			border-radius: 5px;
			border: solid 1px var(--color-border);
			transition: border-color 0.2s, border-width 0.2s;
			cursor: pointer;
		}

		label {
			display: block;
		}

		input {
			display: none;
		}

		input:checked + label &-item {
			border-color: var(--color-highlight);
		}
	}
}

@keyframes spin {
	from {
		transform: rotate(0deg);
	}
	to {
		transform: rotate(360deg);
	}
}
</style>
