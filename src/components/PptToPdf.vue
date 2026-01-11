<template>
  <div class="tool-container">
    <div class="placeholder">
      <p>在纯前端环境中进行 PPT 转 PDF 较为复杂且处于实验阶段。</p>
      <p>当前浏览器版本暂不支持此功能。</p>
    </div>
  </div>
</template>

<script>
export default {
  name: 'PptToPdf'
}
</script>

<style scoped>
.placeholder {
  text-align: center;
  padding: 40px;
  color: #888;
  background: #f9f9f9;
  border-radius: 4px;
  margin-top: 20px;
}

@media (max-width: 768px) {
  .placeholder {
    padding: 20px;
  }
}
</style>
