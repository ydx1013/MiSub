<script setup>
import { ref } from 'vue';
import Modal from './Modal.vue';

const props = defineProps({
  show: Boolean,
});

const emit = defineEmits(['update:show', 'import']);

const importText = ref('');

const handleConfirm = () => {
    emit('import', importText.value);
    emit('update:show', false);
    importText.value = '';
};
</script>

<template>
  <Modal :show="show" @update:show="emit('update:show', $event)" @confirm="handleConfirm">
    <template #title><h3 class="text-lg font-bold text-gray-900 dark:text-white">添加资源</h3></template>
    <template #body>
      <p class="text-sm text-gray-500 dark:text-gray-400 mb-4">
        支持混合粘贴以下内容，系统将自动识别分类：
        <ul class="list-disc list-inside mt-2 space-y-1">
          <li>机场订阅链接 (http://, https://)</li>
          <li>节点分享链接 (vmess://, ss://, trojan:// 等)</li>
        </ul>
      </p>
      <textarea 
        v-model="importText"
        rows="8"
        class="w-full text-sm border-gray-300 dark:border-gray-600 bg-gray-100 dark:bg-gray-700 p-2 rounded-lg focus:outline-hidden focus:ring-2 focus:ring-indigo-500 font-mono dark:text-white"
        placeholder="在此粘贴链接，每行一个..."
      ></textarea>
    </template>
  </Modal>
</template>