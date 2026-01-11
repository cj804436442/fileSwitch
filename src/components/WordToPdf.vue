<template>
  <div class="tool-container">
    <div class="upload-section">
      <input 
        type="file" 
        accept=".docx" 
        @change="handleFileChange"
        ref="fileInput"
      />
      <p v-if="!wordFile">选择要转换的 Word (.docx) 文件</p>
      <p v-else>已选择: {{ wordFile.name }}</p>
    </div>

    <div v-show="wordFile" class="preview-container">
      <h3>预览</h3>
      <div id="word-preview" class="document-preview"></div>
    </div>

    <div class="actions" v-if="wordFile">
      <button @click="generatePdf" :disabled="generating">
        {{ generating ? '生成 PDF 中...' : '下载 PDF' }}
      </button>
    </div>
  </div>
</template>

<script>
import { renderAsync } from 'docx-preview';
import html2pdf from 'html2pdf.js';

export default {
  name: 'WordToPdf',
  data() {
    return {
      wordFile: null,
      generating: false
    };
  },
  methods: {
    handleFileChange(event) {
      const file = event.target.files[0];
      if (!file) return;

      this.wordFile = file;
      
      const reader = new FileReader();
      reader.onload = (e) => {
        const arrayBuffer = e.target.result;
        const container = document.getElementById('word-preview');
        container.innerHTML = ''; // Clear previous content
        
        renderAsync(arrayBuffer, container, undefined, {
          inWrapper: false,
          ignoreWidth: false,
          ignoreHeight: false,
          ignoreFonts: false,
          breakPages: true,
          ignoreLastRenderedPageBreak: false,
          experimental: true,
          trimXmlDeclaration: true,
          useBase64URL: true
        })
          .then(() => {
            console.log('docx rendered');
          })
          .catch(error => {
            console.error('Error rendering docx:', error);
            alert('Error parsing Word file');
          });
      };
      reader.readAsArrayBuffer(file);
    },
    generatePdf() {
      if (!this.wordFile) return;
      
      this.generating = true;
      
      const element = document.getElementById('word-preview');
      
      // Save original styles
      const originalMaxHeight = element.style.maxHeight;
      const originalOverflow = element.style.overflow;
      const originalHeight = element.style.height;
      const originalWidth = element.style.width;
      const originalPadding = element.style.padding;
      const originalMargin = element.style.margin;
      
      // Temporarily remove constraints to render full content
      element.style.maxHeight = 'none';
      element.style.overflow = 'visible';
      element.style.height = 'auto';
      // Force width to match A4 to ensure it fits within PDF
      // A4 is ~210mm. docx-preview usually renders at 96dpi (approx 794px for 210mm)
      // Setting explicit width helps html2canvas capture correct viewport
      element.style.width = '794px'; 
      element.style.padding = '0';
      element.style.margin = '0';
      
      const opt = {
        margin: [0, 0, 0, 0], // Remove PDF margins as docx-preview handles page margins
        filename: this.wordFile.name.replace('.docx', '.pdf'),
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { 
          scale: 2,
          useCORS: true,
          letterRendering: true,
          scrollY: 0,
          windowWidth: 794 // Match the container width
        },
        jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' },
        pagebreak: { mode: ['avoid-all', 'css', 'legacy'] }
      };

      html2pdf().set(opt).from(element).save()
        .then(() => {
          this.generating = false;
          // Restore styles
          element.style.maxHeight = originalMaxHeight;
          element.style.overflow = originalOverflow;
          element.style.height = originalHeight;
          element.style.width = originalWidth;
          element.style.padding = originalPadding;
          element.style.margin = originalMargin;
        })
        .catch(err => {
          console.error('PDF generation error:', err);
          alert('Error generating PDF');
          this.generating = false;
          // Restore styles
          element.style.maxHeight = originalMaxHeight;
          element.style.overflow = originalOverflow;
          element.style.height = originalHeight;
          element.style.width = originalWidth;
          element.style.padding = originalPadding;
          element.style.margin = originalMargin;
        });
    }
  }
};
</script>

<style scoped>
.tool-container {
  padding: 10px;
}

h2 {
  text-align: center;
  color: #555;
  margin-bottom: 20px;
}

.upload-section {
  border: 2px dashed #ccc;
  padding: 40px;
  text-align: center;
  margin-bottom: 20px;
  border-radius: 4px;
}

.preview-container {
  margin-bottom: 20px;
}

.document-preview {
  border: 1px solid #ddd;
  padding: 40px;
  background: white;
  min-height: 400px;
  max-height: 600px;
  overflow-y: auto;
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
}

/* Add some basic styles for the previewed content */
.document-preview >>> p {
  margin-bottom: 1em;
  line-height: 1.5;
}

.document-preview >>> h1, 
.document-preview >>> h2, 
.document-preview >>> h3 {
  margin-top: 1.5em;
  margin-bottom: 0.5em;
}

.document-preview >>> table {
  border-collapse: collapse;
  width: 100%;
}

.document-preview >>> td, 
.document-preview >>> th {
  border: 1px solid #ddd;
  padding: 8px;
}

button {
  background-color: #4caf50;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.actions {
  text-align: center;
  padding-top: 20px;
  border-top: 1px solid #eee;
}

.actions button {
  font-size: 16px;
  padding: 12px 24px;
}
</style>
