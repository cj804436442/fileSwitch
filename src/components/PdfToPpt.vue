<template>
  <div class="tool-container">
    <div class="upload-section">
      <input 
        type="file" 
        accept="application/pdf" 
        @change="handleFileChange"
        ref="fileInput"
      />
      <p v-if="!pdfFile">选择要转换的 PDF 文件</p>
      <p v-else>已选择: {{ pdfFile.name }} (共 {{ pageCount }} 页)</p>
    </div>

    <div v-if="pages.length > 0" class="preview-section">
      <div v-for="(page, index) in pages" :key="index" class="image-card">
        <img :src="page.url" :alt="'Page ' + (index + 1)" />
        <div class="image-info">
          <span>第 {{ index + 1 }} 页</span>
        </div>
      </div>
    </div>

    <div class="actions" v-if="pdfFile">
      <button @click="convertToPpt" :disabled="processing">
        {{ processing ? '转换中...' : '转换为 PPT' }}
      </button>
      <p class="note">注意：此工具会将 PDF 页面作为图片转换为 PPT 演示文稿。</p>
    </div>
  </div>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist';
import pdfWorker from 'pdfjs-dist/build/pdf.worker.mjs?url';
import PptxGenJS from 'pptxgenjs';

pdfjsLib.GlobalWorkerOptions.workerSrc = pdfWorker;

export default {
  name: 'PdfToPpt',
  data() {
    return {
      pdfFile: null,
      pageCount: 0,
      pages: [], // Preview pages
      processing: false
    };
  },
  methods: {
    async handleFileChange(event) {
      const file = event.target.files[0];
      if (!file || file.type !== 'application/pdf') return;

      this.pdfFile = file;
      this.pages = [];
      this.pageCount = 0;
      
      const arrayBuffer = await file.arrayBuffer();
      
      try {
        const loadingTask = pdfjsLib.getDocument({ data: arrayBuffer });
        const pdf = await loadingTask.promise;
        this.pageCount = pdf.numPages;
        
        // Render first page for preview
        const page = await pdf.getPage(1);
        const scale = 0.5; // Preview scale
        const viewport = page.getViewport({ scale });
        
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        canvas.height = viewport.height;
        canvas.width = viewport.width;

        await page.render({
          canvasContext: context,
          viewport: viewport
        }).promise;

        this.pages.push({
          url: canvas.toDataURL('image/jpeg'),
          pageNumber: 1
        });

      } catch (error) {
        console.error('Error loading PDF:', error);
        alert('Error loading PDF file');
      }
    },
    async convertToPpt() {
      if (!this.pdfFile) return;
      
      this.processing = true;
      
      try {
        const arrayBuffer = await this.pdfFile.arrayBuffer();
        const loadingTask = pdfjsLib.getDocument({ data: arrayBuffer });
        const pdf = await loadingTask.promise;
        
        const pptx = new PptxGenJS();
        
        // Get dimensions from first page to set layout
        // PDF units are usually points (1/72 inch)
        // PptxGenJS uses inches by default
        const firstPage = await pdf.getPage(1);
        const layoutViewport = firstPage.getViewport({ scale: 1 });
        const widthInches = layoutViewport.width / 72;
        const heightInches = layoutViewport.height / 72;
        
        pptx.defineLayout({ name: 'PDF_LAYOUT', width: widthInches, height: heightInches });
        pptx.layout = 'PDF_LAYOUT';
        
        for (let i = 1; i <= pdf.numPages; i++) {
          const page = await pdf.getPage(i);
          // High quality for export
          const scale = 2.0; 
          const viewport = page.getViewport({ scale });
          
          const canvas = document.createElement('canvas');
          const context = canvas.getContext('2d');
          canvas.height = viewport.height;
          canvas.width = viewport.width;

          await page.render({
            canvasContext: context,
            viewport: viewport
          }).promise;
          
          const imgData = canvas.toDataURL('image/jpeg', 0.95);
          
          // Create slide
          const slide = pptx.addSlide();
          
          // Add image to slide, filling it completely since aspect ratio matches
          slide.addImage({
            data: imgData,
            x: 0,
            y: 0,
            w: '100%',
            h: '100%'
          });
        }
        
        await pptx.writeFile({ fileName: this.pdfFile.name.replace('.pdf', '.pptx') });
        
      } catch (error) {
        console.error('Error converting PDF to PPT:', error);
        alert('Error converting PDF to PowerPoint');
      } finally {
        this.processing = false;
      }
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

.preview-section {
  display: flex;
  justify-content: center;
  margin-bottom: 20px;
}

.image-card {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 10px;
  background: #fff;
  text-align: center;
}

.image-card img {
  max-width: 200px;
  display: block;
  margin-bottom: 10px;
  border: 1px solid #eee;
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

.note {
  font-size: 12px;
  color: #666;
  margin-top: 10px;
}

@media (max-width: 768px) {
  .upload-section {
    padding: 20px;
  }
}
</style>
