<template>
  <div class="tool-container">
    <div class="actions">
      <button type="button" class="open-task-btn" @click="openTaskTracking">
        打开任务追踪
      </button>
    </div>

    <div class="placeholder">
      <p>在纯前端环境中进行 PPT 转 PDF 较为复杂且处于实验阶段。</p>
      <p>当前浏览器版本暂不支持此功能。</p>
    </div>

    <TaskTrackingPlugin ref="taskTracking" :show-floating-button="false" />
  </div>
</template>

<script>
import TaskTrackingPlugin from './TaskTrackingPlugin.vue';

export default {
  name: 'PptToPdf',
  components: {
    TaskTrackingPlugin
  },
  methods: {
    openTaskTracking() {
      this.$refs.taskTracking?.openPanel();
    }
  }
}
</script>

<style scoped>
.actions {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 16px;
}

.open-task-btn {
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 10px 14px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.2s ease;
}

.open-task-btn:hover {
  background: #1d4ed8;
}

.placeholder {
  text-align: center;
  padding: 40px;
  color: #888;
  background: #f9f9f9;
  border-radius: 4px;
  margin-top: 20px;
}

@media (max-width: 768px) {
  .actions {
    justify-content: center;
    margin-bottom: 12px;
  }

  .placeholder {
    padding: 20px;
  }
}
</style>
