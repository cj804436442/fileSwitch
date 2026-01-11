<template>
  <div class="tool-container">
    <div class="upload-section">
      <input 
        type="file" 
        accept=".pdf" 
        multiple 
        @change="handleFileChange" 
        ref="fileInput"
      />
      <p class="hint">选择多个 PDF 文件进行合并</p>
    </div>

    <div v-if="pdfFiles.length > 0" class="file-list">
      <h3>要合并的文件 ({{ pdfFiles.length }})</h3>
      <ul>
        <li v-for="(file, index) in pdfFiles" :key="index + '-' + file.name">
          <span class="file-name">{{ index + 1 }}. {{ file.name }}</span>
          <div class="file-actions">
            <button @click="moveUp(index)" :disabled="index === 0">↑</button>
            <button @click="moveDown(index)" :disabled="index === pdfFiles.length - 1">↓</button>
            <button @click="removeFile(index)" class="remove-btn">✕</button>
          </div>
        </li>
      </ul>
      
      <div class="actions">
        <button @click="mergePdfs" :disabled="processing || pdfFiles.length < 2" class="primary-btn">
          {{ processing ? '合并中...' : '合并 PDF' }}
        </button>
        <button @click="clearAll" class="secondary-btn">清空所有</button>
      </div>
    </div>
  </div>
</template>

<script>
import { PDFDocument } from 'pdf-lib';
import { saveAs } from 'file-saver';

export default {
  name: 'PdfMerge',
  data() {
    return {
      pdfFiles: [],
      processing: false
    };
  },
  methods: {
    handleFileChange(event) {
      const files = Array.from(event.target.files);
      if (files.length > 0) {
        // Append new files to existing list
        this.pdfFiles = [...this.pdfFiles, ...files];
      }
      // Reset input so the same files can be selected again if needed
      event.target.value = '';
    },
    moveUp(index) {
      if (index > 0) {
        const item = this.pdfFiles[index];
        this.pdfFiles.splice(index, 1);
        this.pdfFiles.splice(index - 1, 0, item);
      }
    },
    moveDown(index) {
      if (index < this.pdfFiles.length - 1) {
        const item = this.pdfFiles[index];
        this.pdfFiles.splice(index, 1);
        this.pdfFiles.splice(index + 1, 0, item);
      }
    },
    removeFile(index) {
      this.pdfFiles.splice(index, 1);
    },
    clearAll() {
      this.pdfFiles = [];
    },
    async mergePdfs() {
      if (this.pdfFiles.length < 2) return;
      
      this.processing = true;
      try {
        const mergedPdf = await PDFDocument.create();
        
        for (const file of this.pdfFiles) {
          const arrayBuffer = await file.arrayBuffer();
          const pdfDoc = await PDFDocument.load(arrayBuffer);
          const copiedPages = await mergedPdf.copyPages(pdfDoc, pdfDoc.getPageIndices());
          copiedPages.forEach((page) => mergedPdf.addPage(page));
        }
        
        const mergedPdfBytes = await mergedPdf.save();
        const blob = new Blob([mergedPdfBytes], { type: 'application/pdf' });
        saveAs(blob, 'merged-document.pdf');
        
      } catch (error) {
        console.error('Error merging PDFs:', error);
        alert('Failed to merge PDFs. Please ensure all files are valid PDFs.');
      } finally {
        this.processing = false;
      }
    }
  }
};
</script>

<style scoped>
.tool-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.upload-section {
  border: 2px dashed #ccc;
  padding: 40px;
  text-align: center;
  border-radius: 8px;
  margin-bottom: 30px;
  background-color: #fafafa;
}

.hint {
  color: #666;
  margin-top: 10px;
}

.file-list ul {
  list-style: none;
  padding: 0;
  margin-bottom: 20px;
  border: 1px solid #eee;
  border-radius: 4px;
}

.file-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 15px;
  border-bottom: 1px solid #eee;
  background-color: white;
}

.file-list li:last-child {
  border-bottom: none;
}

.file-name {
  font-size: 14px;
  word-break: break-all;
}

.file-actions button {
  margin-left: 5px;
  padding: 4px 8px;
  cursor: pointer;
  background-color: #f0f0f0;
  border: 1px solid #ccc;
  border-radius: 4px;
}

.file-actions button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.file-actions .remove-btn {
  background-color: #ffeff0;
  border-color: #ffcccc;
  color: #d32f2f;
}

.actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.primary-btn {
  background-color: #4caf50;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

.primary-btn:disabled {
  background-color: #a5d6a7;
  cursor: not-allowed;
}

.secondary-btn {
  background-color: white;
  color: #666;
  padding: 10px 20px;
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: pointer;
}

@media (max-width: 768px) {
  .upload-section {
    padding: 20px;
  }
  
  .file-list li {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .file-actions {
    margin-top: 10px;
    align-self: flex-end;
  }
}
</style>
