<script setup>
import { ref, watch, computed } from 'vue';
import Modal from './Modal.vue';

const props = defineProps({
  show: Boolean,
  profile: Object,
  isNew: Boolean,
  allSubscriptions: Array,
  allManualNodes: Array,
});

const emit = defineEmits(['update:show', 'save']);

const localProfile = ref({});
const subscriptionSearchTerm = ref('');
const nodeSearchTerm = ref('');
const subscriptionNodes = ref({}); // { subId: [{ name: 'Node 1', original: '...' }, ...] }
const loadingNodes = ref({}); // { subId: true/false }
const activeSubscriptionId = ref(null); // Currently selected subscription for node management
const nodeFilterSearchTerm = ref(''); // Search term for nodes within a subscription

// 国家/地区代码到旗帜和中文名称的映射
const countryCodeMap = {
  'hk': ['🇭🇰', '香港'],
  'tw': ['🇹🇼', '台湾', '臺灣'],
  'sg': ['🇸🇬', '新加坡', '狮城'],
  'jp': ['🇯🇵', '日本'],
  'us': ['🇺🇸', '美国', '美國'],
  'kr': ['🇰🇷', '韩国', '韓國'],
  'gb': ['🇬🇧', '英国', '英國'],
  'de': ['🇩🇪', '德国', '德國'],
  'fr': ['🇫🇷', '法国', '法國'],
  'ca': ['🇨🇦', '加拿大'],
  'au': ['🇦🇺', '澳大利亚', '澳洲', '澳大利亞'],
  'cn': ['🇨🇳', '中国', '大陸', '内地'],
  'my': ['🇲🇾', '马来西亚', '馬來西亞'],
  'th': ['🇹🇭', '泰国', '泰國'],
  'vn': ['🇻🇳', '越南'],
  'ph': ['🇵🇭', '菲律宾', '菲律賓'],
  'id': ['🇮🇩', '印度尼西亚', '印尼'],
  'in': ['🇮🇳', '印度'],
  'pk': ['🇵🇰', '巴基斯坦'],
  'bd': ['🇧🇩', '孟加拉国', '孟加拉國'],
  'ae': ['🇦🇪', '阿联酋', '阿聯酋'],
  'sa': ['🇸🇦', '沙特阿拉伯'],
  'tr': ['🇹🇷', '土耳其'],
  'ru': ['🇷🇺', '俄罗斯', '俄羅斯'],
  'br': ['🇧🇷', '巴西'],
  'mx': ['🇲🇽', '墨西哥'],
  'ar': ['🇦🇷', '阿根廷'],
  'cl': ['🇨🇱', '智利'],
  'za': ['🇿🇦', '南非'],
  'eg': ['🇪🇬', '埃及'],
  'ng': ['🇳🇬', '尼日利亚', '尼日利亞'],
  'ke': ['🇰🇪', '肯尼亚', '肯尼亞'],
  'il': ['🇮🇱', '以色列'],
  'ir': ['🇮🇷', '伊朗'],
  'iq': ['🇮🇶', '伊拉克'],
  'ua': ['🇺🇦', '乌克兰', '烏克蘭'],
  'pl': ['🇵🇱', '波兰', '波蘭'],
  'cz': ['🇨🇿', '捷克'],
  'hu': ['🇭🇺', '匈牙利'],
  'ro': ['🇷🇴', '罗马尼亚', '羅馬尼亞'],
  'gr': ['🇬🇷', '希腊', '希臘'],
  'pt': ['🇵🇹', '葡萄牙'],
  'es': ['🇪🇸', '西班牙'],
  'it': ['🇮🇹', '意大利'],
  'nl': ['🇳🇱', '荷兰', '荷蘭'],
  'be': ['🇧🇪', '比利时', '比利時'],
  'se': ['🇸🇪', '瑞典'],
  'no': ['🇳🇴', '挪威'],
  'dk': ['🇩🇰', '丹麦', '丹麥'],
  'fi': ['🇫🇮', '芬兰', '芬蘭'],
  'ch': ['🇨🇭', '瑞士'],
  'at': ['🇦🇹', '奥地利', '奧地利'],
  'ie': ['🇮🇪', '爱尔兰', '愛爾蘭'],
  'nz': ['🇳🇿', '新西兰', '紐西蘭'],
};

const filteredSubscriptions = computed(() => {
  if (!subscriptionSearchTerm.value) {
    return props.allSubscriptions;
  }
  const lowerCaseSearchTerm = subscriptionSearchTerm.value.toLowerCase();
  const alternativeTerms = countryCodeMap[lowerCaseSearchTerm] || [];

  return props.allSubscriptions.filter(sub => {
    const subNameLower = sub.name ? sub.name.toLowerCase() : '';

    if (subNameLower.includes(lowerCaseSearchTerm)) {
      return true;
    }

    for (const altTerm of alternativeTerms) {
      if (subNameLower.includes(altTerm.toLowerCase())) {
        return true;
      }
    }
    return false;
  });
});

const filteredManualNodes = computed(() => {
  if (!nodeSearchTerm.value) {
    return props.allManualNodes;
  }
  const lowerCaseSearchTerm = nodeSearchTerm.value.toLowerCase();
  const alternativeTerms = countryCodeMap[lowerCaseSearchTerm] || [];

  return props.allManualNodes.filter(node => {
    const nodeNameLower = node.name ? node.name.toLowerCase() : '';

    if (nodeNameLower.includes(lowerCaseSearchTerm)) {
      return true;
    }

    for (const altTerm of alternativeTerms) {
      if (nodeNameLower.includes(altTerm.toLowerCase())) {
        return true;
      }
    }
    return false;
  });
});

const activeSubscription = computed(() => {
    if (!activeSubscriptionId.value) return null;
    return props.allSubscriptions.find(s => s.id === activeSubscriptionId.value);
});

const filteredSubscriptionNodes = computed(() => {
    if (!activeSubscriptionId.value) return [];
    const nodes = subscriptionNodes.value[activeSubscriptionId.value] || [];
    if (!nodeFilterSearchTerm.value) return nodes;
    
    const term = nodeFilterSearchTerm.value.toLowerCase();
    return nodes.filter(node => node.name.toLowerCase().includes(term));
});

const toggleSubscriptionExpand = async (sub) => {
  expandedSubscriptions.value[sub.id] = !expandedSubscriptions.value[sub.id];
  if (expandedSubscriptions.value[sub.id] && !subscriptionNodes.value[sub.id]) {
    await fetchNodes(sub);
  }
};

const selectSubscriptionForNodes = async (sub) => {
    activeSubscriptionId.value = sub.id;
    if (!subscriptionNodes.value[sub.id]) {
        await fetchNodes(sub);
    }
};

const fetchNodes = async (sub) => {
  if (!sub.id) return;
  loadingNodes.value[sub.id] = true;
  try {
    // Use cached nodes instead of fetching external URL
    const response = await fetch('/api/get_cached_nodes', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ id: sub.id })
    });
    if (!response.ok) throw new Error('Fetch failed');
    
    const data = await response.json();
    if (data.error) throw new Error(data.error);
    
    subscriptionNodes.value[sub.id] = data.nodes || [];
  } catch (error) {
    console.error('Failed to fetch nodes', error);
    subscriptionNodes.value[sub.id] = []; // Error state
  } finally {
    loadingNodes.value[sub.id] = false;
  }
};

const isNodeSelected = (subId, nodeName) => {
    const filters = localProfile.value.nodeFilters?.[subId];
    if (!filters) return true; // Default include all (no filter means include all)
    
    if (filters.mode === 'exclude') {
        return !filters.list.includes(nodeName);
    } else if (filters.mode === 'include') {
        return filters.list.includes(nodeName);
    }
    return true;
};

const toggleNodeSelection = (subId, nodeName) => {
    if (!localProfile.value.nodeFilters) localProfile.value.nodeFilters = {};
    
    // Initialize if not exists. Default to 'exclude' mode (blacklist)
    if (!localProfile.value.nodeFilters[subId]) {
        localProfile.value.nodeFilters[subId] = { mode: 'exclude', list: [] };
    }
    
    const filters = localProfile.value.nodeFilters[subId];
    
    if (filters.mode === 'exclude') {
        // In exclude mode:
        // If currently selected (not in list), we want to deselect (add to list)
        // If currently deselected (in list), we want to select (remove from list)
        const index = filters.list.indexOf(nodeName);
        if (index === -1) {
            filters.list.push(nodeName); // Exclude it
        } else {
            filters.list.splice(index, 1); // Include it (remove from exclude list)
        }
    } else {
        // In include mode:
        // If currently selected (in list), we want to deselect (remove from list)
        // If currently deselected (not in list), we want to select (add to list)
        const index = filters.list.indexOf(nodeName);
        if (index === -1) {
            filters.list.push(nodeName); // Include it
        } else {
            filters.list.splice(index, 1); // Exclude it (remove from include list)
        }
    }
};

const toggleFilterMode = (subId) => {
    if (!localProfile.value.nodeFilters) localProfile.value.nodeFilters = {};
    if (!localProfile.value.nodeFilters[subId]) {
        localProfile.value.nodeFilters[subId] = { mode: 'include', list: [] }; // Switch to include (whitelist) empty = none
    } else {
        const filters = localProfile.value.nodeFilters[subId];
        // Toggle mode
        filters.mode = filters.mode === 'exclude' ? 'include' : 'exclude';
        // Reset list when switching modes? Or try to invert?
        // Inverting is hard because we need the full list of nodes.
        // Let's just reset list for simplicity or keep it empty.
        filters.list = [];
    }
};

watch(() => props.profile, (newProfile) => {
  if (newProfile) {
    const profileCopy = JSON.parse(JSON.stringify(newProfile));
    // Format date for input[type=date]
    if (profileCopy.expiresAt) {
      try {
        profileCopy.expiresAt = new Date(profileCopy.expiresAt).toISOString().split('T')[0];
      } catch (e) {
        console.error("Error parsing expiresAt date:", e);
        profileCopy.expiresAt = '';
      }
    }
    // 初始化前缀设置
    if (!profileCopy.prefixSettings) {
      profileCopy.prefixSettings = {
        enableManualNodes: null,
        enableSubscriptions: null,
        manualNodePrefix: ''
      };
    }
    // 初始化节点筛选设置
    if (!profileCopy.nodeFilters) {
      profileCopy.nodeFilters = {}; // { subId: { mode: 'exclude'|'include', rules: ['keyword1', 'keyword2'] } }
    }
    localProfile.value = profileCopy;
  } else {
    localProfile.value = { 
      name: '', 
      enabled: true, 
      subscriptions: [], 
      manualNodes: [], 
      customId: '', 
      expiresAt: '',
      prefixSettings: {
        enableManualNodes: null,
        enableSubscriptions: null,
        manualNodePrefix: ''
      },
      nodeFilters: {}
    };
  }
}, { deep: true, immediate: true });

const handleConfirm = () => {
  const profileToSave = JSON.parse(JSON.stringify(localProfile.value));
  if (profileToSave.expiresAt) {
    try {
      // Set time to the end of the selected day in local time, then convert to ISO string
      const date = new Date(profileToSave.expiresAt);
      date.setHours(23, 59, 59, 999);
      profileToSave.expiresAt = date.toISOString();
    } catch (e) {
      console.error("Error processing expiresAt date:", e);
      // Decide how to handle error: save as is, or clear it
      profileToSave.expiresAt = ''; 
    }
  }
  emit('save', profileToSave);
};

const toggleSelection = (listName, id) => {
    const list = localProfile.value[listName];
    const index = list.indexOf(id);
    if (index > -1) {
        list.splice(index, 1);
    } else {
        list.push(id);
    }
};

const handleSelectAll = (listName, sourceArray) => {
    const currentSelection = new Set(localProfile.value[listName]);
    sourceArray.forEach(item => currentSelection.add(item.id));
    localProfile.value[listName] = Array.from(currentSelection);
};

const handleDeselectAll = (listName, sourceArray) => {
    const sourceIds = sourceArray.map(item => item.id);
    localProfile.value[listName] = localProfile.value[listName].filter(id => !sourceIds.includes(id));
};

const handleSelectAllNodes = () => {
    if (!activeSubscriptionId.value) return;
    const subId = activeSubscriptionId.value;
    const nodes = filteredSubscriptionNodes.value;
    
    if (!localProfile.value.nodeFilters) localProfile.value.nodeFilters = {};
    if (!localProfile.value.nodeFilters[subId]) localProfile.value.nodeFilters[subId] = { mode: 'exclude', list: [] };
    
    const filter = localProfile.value.nodeFilters[subId];
    
    // Select All means:
    // If mode is 'include': Add all visible nodes to list
    // If mode is 'exclude': Remove all visible nodes from list
    
    nodes.forEach(node => {
        const index = filter.list.indexOf(node.name);
        if (filter.mode === 'include') {
            if (index === -1) filter.list.push(node.name);
        } else {
            if (index !== -1) filter.list.splice(index, 1);
        }
    });
};

const handleDeselectAllNodes = () => {
    if (!activeSubscriptionId.value) return;
    const subId = activeSubscriptionId.value;
    const nodes = filteredSubscriptionNodes.value;
    
    if (!localProfile.value.nodeFilters) localProfile.value.nodeFilters = {};
    if (!localProfile.value.nodeFilters[subId]) localProfile.value.nodeFilters[subId] = { mode: 'exclude', list: [] };
    
    const filter = localProfile.value.nodeFilters[subId];
    
    // Deselect All means:
    // If mode is 'include': Remove all visible nodes from list
    // If mode is 'exclude': Add all visible nodes to list
    
    nodes.forEach(node => {
        const index = filter.list.indexOf(node.name);
        if (filter.mode === 'include') {
            if (index !== -1) filter.list.splice(index, 1);
        } else {
            if (index === -1) filter.list.push(node.name);
        }
    });
};

</script>

<template>
  <Modal :show="show" @update:show="emit('update:show', $event)" @confirm="handleConfirm" size="5xl">
    <template #title>
      <h3 class="text-lg font-bold text-gray-800 dark:text-white">
        {{ isNew ? '新增订阅组' : '编辑订阅组' }}
      </h3>
    </template>
    <template #body>
      <div v-if="localProfile" class="space-y-6">
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label for="profile-name" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
                订阅组名称
              </label>
              <input
                type="text"
                id="profile-name"
                v-model="localProfile.name"
                placeholder="例如：家庭共享"
                class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded-md shadow-xs focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"
              >
            </div>
            <div>
              <label for="profile-custom-id" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
                自定义 ID (可选)
              </label>
              <input
                type="text"
                id="profile-custom-id"
                v-model="localProfile.customId"
                placeholder="如: home, game (限字母、数字、-、_)"
                class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded-md shadow-xs focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"
              >
               <p class="text-xs text-gray-400 mt-1">设置后，订阅链接会更短，如 /token/home</p>
            </div>
            <div>
              <label for="profile-subconverter" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
                自定义后端 (可选)
              </label>
              <input
                type="text"
                id="profile-subconverter"
                v-model="localProfile.subConverter"
                placeholder="留空则使用全局设置"
                class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded-md shadow-xs focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"
              >
              <p class="text-xs text-gray-400 mt-1">为此订阅组指定一个独立的 SubConverter 后端地址。</p>
            </div>
            <div>
              <label for="profile-subconfig" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
                自定义远程配置 (可选)
              </label>
              <input
                type="text"
                id="profile-subconfig"
                v-model="localProfile.subConfig"
                placeholder="留空则使用全局设置"
                class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded-md shadow-xs focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"
              >
              <p class="text-xs text-gray-400 mt-1">为此订阅组指定一个独立的 Subconverter 配置文件。</p>
            </div>
            <div>
              <label for="profile-expires-at" class="block text-sm font-medium text-gray-700 dark:text-gray-300">
                到期时间 (可选)
              </label>
              <input
                type="date"
                id="profile-expires-at"
                v-model="localProfile.expiresAt"
                class="mt-1 block w-full px-3 py-2 bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded-md shadow-xs focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 sm:text-sm dark:text-white"
              >
              <p class="text-xs text-gray-400 mt-1">设置此订阅组的到期时间，到期后将返回默认节点。</p>
            </div>
            
            <!-- 前缀设置部分 -->
            <div>
              <label class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-3">前缀设置 (可选)</label>
              <div class="space-y-3 bg-gray-50 dark:bg-gray-800/50 rounded-lg p-3">
                <div class="flex items-center justify-between">
                  <div>
                    <p class="text-sm font-medium text-gray-700 dark:text-gray-300">手动节点前缀</p>
                    <p class="text-xs text-gray-500 dark:text-gray-400">覆盖全局设置，控制是否为手动节点添加前缀</p>
                  </div>
                  <select 
                    v-model="localProfile.prefixSettings.enableManualNodes" 
                    class="text-sm bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded px-2 py-1 focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 dark:text-white"
                  >
                    <option :value="null">使用全局设置</option>
                    <option :value="true">启用</option>
                    <option :value="false">禁用</option>
                  </select>
                </div>
                
                <div v-if="localProfile.prefixSettings.enableManualNodes === true" class="ml-4">
                  <label class="block text-xs font-medium text-gray-600 dark:text-gray-400 mb-1">自定义手动节点前缀</label>
                  <input 
                    type="text" 
                    v-model="localProfile.prefixSettings.manualNodePrefix" 
                    placeholder="留空使用全局设置"
                    class="block w-full px-2 py-1 text-sm bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded shadow-xs focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 dark:text-white"
                  >
                </div>
                
                <div class="flex items-center justify-between">
                  <div>
                    <p class="text-sm font-medium text-gray-700 dark:text-gray-300">机场订阅前缀</p>
                    <p class="text-xs text-gray-500 dark:text-gray-400">覆盖全局设置，控制是否为订阅节点添加前缀</p>
                  </div>
                  <select 
                    v-model="localProfile.prefixSettings.enableSubscriptions" 
                    class="text-sm bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded px-2 py-1 focus:outline-hidden focus:ring-indigo-500 focus:border-indigo-500 dark:text-white"
                  >
                    <option :value="null">使用全局设置</option>
                    <option :value="true">启用</option>
                    <option :value="false">禁用</option>
                  </select>
                </div>
              </div>
              <p class="text-xs text-gray-400 mt-1">单独为此订阅组配置前缀设置，优先级高于全局设置。</p>
            </div>
        </div>

        <!-- Subscription Management Section (Two Columns) -->
        <div class="grid grid-cols-1 lg:grid-cols-3 gap-6 h-[500px]">
            <!-- Left Column: Subscription List -->
            <div class="lg:col-span-1 flex flex-col h-full border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden">
                <div class="p-3 bg-gray-50 dark:bg-gray-800 border-b border-gray-200 dark:border-gray-700">
                    <h4 class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">选择机场订阅</h4>
                    <div class="relative">
                        <input
                            type="text"
                            v-model="subscriptionSearchTerm"
                            placeholder="搜索订阅..."
                            class="w-full pl-8 pr-3 py-1.5 text-xs bg-white dark:bg-gray-700 border border-gray-300 dark:border-gray-600 rounded-md focus:outline-hidden focus:ring-1 focus:ring-indigo-500"
                        />
                        <svg class="absolute left-2.5 top-1/2 -translate-y-1/2 h-3.5 w-3.5 text-gray-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" /></svg>
                    </div>
                </div>
                <div class="flex-1 overflow-y-auto p-2 space-y-1 bg-white dark:bg-gray-900">
                    <div v-if="filteredSubscriptions.length === 0" class="text-center text-gray-500 text-xs py-4">
                        无匹配订阅
                    </div>
                    <div 
                        v-for="sub in filteredSubscriptions" 
                        :key="sub.id" 
                        class="flex items-center justify-between p-2 rounded-md cursor-pointer transition-colors group"
                        :class="activeSubscriptionId === sub.id ? 'bg-indigo-50 dark:bg-indigo-900/30' : 'hover:bg-gray-50 dark:hover:bg-gray-800'"
                        @click="selectSubscriptionForNodes(sub)"
                    >
                        <div class="flex items-center space-x-3 overflow-hidden">
                            <input
                                type="checkbox"
                                :checked="localProfile.subscriptions?.includes(sub.id)"
                                @change.stop="toggleSelection('subscriptions', sub.id)"
                                class="h-4 w-4 rounded-sm border-gray-300 text-indigo-600 focus:ring-indigo-500 shrink-0"
                            />
                            <span class="text-sm text-gray-700 dark:text-gray-200 truncate select-none">{{ sub.name || '未命名订阅' }}</span>
                        </div>
                        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-gray-400 group-hover:text-indigo-500" :class="{'text-indigo-600': activeSubscriptionId === sub.id}" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7" />
                        </svg>
                    </div>
                </div>
                <div class="p-2 bg-gray-50 dark:bg-gray-800 border-t border-gray-200 dark:border-gray-700 flex justify-between text-xs">
                    <button @click="handleSelectAll('subscriptions', filteredSubscriptions)" class="text-indigo-600 hover:underline">全选</button>
                    <button @click="handleDeselectAll('subscriptions', filteredSubscriptions)" class="text-indigo-600 hover:underline">全不选</button>
                </div>
            </div>

            <!-- Right Column: Node Management -->
            <div class="lg:col-span-2 flex flex-col h-full border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden bg-white dark:bg-gray-900">
                <div v-if="!activeSubscriptionId" class="flex-1 flex flex-col items-center justify-center text-gray-400 p-6 text-center">
                    <svg xmlns="http://www.w3.org/2000/svg" class="h-12 w-12 mb-3 opacity-50" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10" />
                    </svg>
                    <p class="text-sm">请在左侧选择一个订阅以管理其节点</p>
                    <p class="text-xs mt-1 opacity-70">您可以筛选特定节点包含在订阅组中</p>
                </div>
                
                <div v-else class="flex flex-col h-full">
                    <!-- Header -->
                    <div class="p-3 bg-gray-50 dark:bg-gray-800 border-b border-gray-200 dark:border-gray-700 flex justify-between items-center shrink-0">
                        <div>
                            <h4 class="text-sm font-bold text-gray-800 dark:text-white flex items-center gap-2">
                                {{ activeSubscription?.name }}
                                <span v-if="loadingNodes[activeSubscriptionId]" class="text-xs font-normal text-gray-500 animate-pulse">(加载中...)</span>
                                <span v-else class="text-xs font-normal text-gray-500">({{ subscriptionNodes[activeSubscriptionId]?.length || 0 }} 节点)</span>
                            </h4>
                        </div>
                        <div class="flex items-center gap-2">
                            <span class="text-xs text-gray-500">模式:</span>
                            <button 
                                @click="toggleFilterMode(activeSubscriptionId)" 
                                class="text-xs px-2 py-1 rounded-md transition-colors border"
                                :class="localProfile.nodeFilters?.[activeSubscriptionId]?.mode === 'include' 
                                    ? 'bg-green-100 text-green-700 border-green-200 dark:bg-green-900/30 dark:text-green-300 dark:border-green-800' 
                                    : 'bg-gray-100 text-gray-700 border-gray-200 dark:bg-gray-700 dark:text-gray-300 dark:border-gray-600'"
                            >
                                {{ localProfile.nodeFilters?.[activeSubscriptionId]?.mode === 'include' ? '白名单 (仅包含选中)' : '黑名单 (排除选中)' }}
                            </button>
                        </div>
                    </div>

                    <!-- Toolbar -->
                    <div class="p-2 border-b border-gray-200 dark:border-gray-700 flex gap-2 items-center bg-white dark:bg-gray-900 shrink-0">
                        <div class="relative flex-1">
                            <input
                                type="text"
                                v-model="nodeFilterSearchTerm"
                                placeholder="搜索节点名称..."
                                class="w-full pl-8 pr-3 py-1.5 text-xs bg-gray-50 dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md focus:outline-hidden focus:ring-1 focus:ring-indigo-500"
                            />
                            <svg class="absolute left-2.5 top-1/2 -translate-y-1/2 h-3.5 w-3.5 text-gray-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" /></svg>
                        </div>
                        <div class="flex gap-1">
                            <button @click="handleSelectAllNodes" class="px-2 py-1.5 text-xs font-medium text-gray-700 dark:text-gray-300 bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 rounded-md transition-colors">全选</button>
                            <button @click="handleDeselectAllNodes" class="px-2 py-1.5 text-xs font-medium text-gray-700 dark:text-gray-300 bg-gray-100 dark:bg-gray-800 hover:bg-gray-200 dark:hover:bg-gray-700 rounded-md transition-colors">全不选</button>
                        </div>
                    </div>

                    <!-- Node List -->
                    <div class="flex-1 overflow-y-auto p-2 bg-gray-50/50 dark:bg-gray-900/50">
                        <div v-if="loadingNodes[activeSubscriptionId]" class="flex flex-col items-center justify-center h-full text-gray-500 space-y-2">
                            <svg class="animate-spin h-6 w-6 text-indigo-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle><path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                            <span class="text-xs">正在解析节点...</span>
                        </div>
                        <div v-else-if="!subscriptionNodes[activeSubscriptionId] || subscriptionNodes[activeSubscriptionId].length === 0" class="flex flex-col items-center justify-center h-full text-gray-500">
                            <span class="text-sm">未找到节点</span>
                            <span class="text-xs mt-1 opacity-70">请检查订阅链接是否有效</span>
                        </div>
                        <div v-else-if="filteredSubscriptionNodes.length === 0" class="flex flex-col items-center justify-center h-full text-gray-500">
                            <span class="text-sm">无匹配节点</span>
                        </div>
                        <div v-else class="grid grid-cols-1 sm:grid-cols-2 gap-2">
                            <div 
                                v-for="(node, idx) in filteredSubscriptionNodes" 
                                :key="idx" 
                                class="flex items-start space-x-2 p-2 rounded-md border border-gray-100 dark:border-gray-800 bg-white dark:bg-gray-800 hover:border-indigo-200 dark:hover:border-indigo-800 transition-colors cursor-pointer"
                                @click="toggleNodeSelection(activeSubscriptionId, node.name)"
                            >
                                <input 
                                    type="checkbox" 
                                    :checked="isNodeSelected(activeSubscriptionId, node.name)"
                                    class="mt-0.5 h-4 w-4 rounded-sm border-gray-300 text-indigo-600 focus:ring-indigo-500 shrink-0 pointer-events-none"
                                >
                                <div class="min-w-0 flex-1">
                                    <p class="text-xs font-medium text-gray-700 dark:text-gray-200 truncate" :title="node.name">{{ node.name }}</p>
                                    <p class="text-[10px] text-gray-400 truncate mt-0.5 font-mono">{{ node.original.split('://')[0] }}</p>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mt-6 pt-6 border-t border-gray-200 dark:border-gray-700">
            <div v-if="allManualNodes.length > 0" class="space-y-2 md:col-span-2">
              <div class="flex justify-between items-center mb-2">
                <h4 class="text-sm font-medium text-gray-700 dark:text-gray-300">选择手动节点</h4>
                 <div class="space-x-2">
                    <button @click="handleSelectAll('manualNodes', filteredManualNodes)" class="text-xs text-indigo-600 hover:underline">全选</button>
                    <button @click="handleDeselectAll('manualNodes', filteredManualNodes)" class="text-xs text-indigo-600 hover:underline">全不选</button>
                </div>
              </div>
              <div class="relative mb-2">
                <input
                  type="text"
                  v-model="nodeSearchTerm"
                  placeholder="搜索节点..."
                  class="w-full pl-9 pr-3 py-1.5 text-sm bg-white dark:bg-gray-800 border border-gray-300 dark:border-gray-600 rounded-md shadow-xs focus:outline-hidden focus:ring-1 focus:ring-indigo-500 focus:border-indigo-500"
                />
                <svg class="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-gray-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" /></svg>
              </div>
               <div class="overflow-y-auto space-y-2 p-3 bg-gray-50 dark:bg-gray-900/50 rounded-lg border dark:border-gray-700 h-48">
                <div v-for="node in filteredManualNodes" :key="node.id">
                  <label class="flex items-center space-x-3 cursor-pointer">
                    <input
                      type="checkbox"
                      :checked="localProfile.manualNodes?.includes(node.id)"
                      @change="toggleSelection('manualNodes', node.id)"
                      class="h-4 w-4 rounded-sm border-gray-300 text-indigo-600 focus:ring-indigo-500"
                    />
                    <span class="text-sm text-gray-800 dark:text-gray-200 truncate" :title="node.name">{{ node.name || '未命名节点' }}</span>
                  </label>
                </div>
                <div v-if="filteredManualNodes.length === 0" class="text-center text-gray-500 text-sm py-4">
                  没有找到匹配的节点。
                </div>
              </div>
            </div>
            <div v-else class="text-center text-sm text-gray-500 p-4 bg-gray-50 dark:bg-gray-900/50 rounded-lg flex items-center justify-center h-full md:col-span-2">
               没有可用的手动节点
            </div>
        </div>

      </div>
    </template>
  </Modal>
</template>