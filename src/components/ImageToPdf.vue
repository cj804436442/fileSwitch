<template>
  <div class="tool-container">
    <div class="upload-section">
      <input 
        type="file" 
        multiple 
        accept="image/*" 
        @change="handleFileChange"
        ref="fileInput"
      />
      <p v-if="images.length === 0">选择要转换的图片</p>
    </div>

    <div v-if="images.length > 0" class="preview-section">
      <div v-for="(img, index) in images" :key="index" class="image-card">
        <img :src="img.url" :alt="img.name" />
        <div class="image-info">
          <span>{{ img.name }}</span>
          <button @click="removeImage(index)">移除</button>
        </div>
      </div>
    </div>

    <div class="actions" v-if="images.length > 0">
      <button @click="generatePDF" :disabled="generating">
        {{ generating ? '生成中...' : '下载 PDF' }}
      </button>
    </div>
  </div>
</template>

<script>
import jsPDF from 'jspdf';

export default {
  data() {
    return {
      images: [],
      generating: false
    };
  },
  methods: {
    handleFileChange(event) {
      const files = Array.from(event.target.files);
      if (!files.length) return;

      files.forEach(file => {
        if (!file.type.startsWith('image/')) return;

        const reader = new FileReader();
        reader.onload = (e) => {
          this.images.push({
            file,
            name: file.name,
            url: e.target.result,
            width: 0,
            height: 0
          });
          
          // Get image dimensions
          const img = new Image();
          img.onload = () => {
            const currentImg = this.images.find(i => i.url === e.target.result);
            if (currentImg) {
              currentImg.width = img.width;
              currentImg.height = img.height;
            }
          };
          img.src = e.target.result;
        };
        reader.readAsDataURL(file);
      });
      
      // Reset input so same files can be selected again if needed
      this.$refs.fileInput.value = '';
    },
    removeImage(index) {
      this.images.splice(index, 1);
    },
    async generatePDF() {
      if (this.images.length === 0) return;
      
      this.generating = true;
      
      try {
        // A4 size in mm: 210 x 297
        const pdf = new jsPDF();
        const pageWidth = pdf.internal.pageSize.getWidth();
        const pageHeight = pdf.internal.pageSize.getHeight();
        
        for (let i = 0; i < this.images.length; i++) {
          const imgData = this.images[i];
          
          if (i > 0) {
            pdf.addPage();
          }
          
          // Calculate dimensions to fit page while maintaining aspect ratio
          const imgRatio = imgData.width / imgData.height;
          const pageRatio = pageWidth / pageHeight;
          
          let renderWidth = pageWidth;
          let renderHeight = pageWidth / imgRatio;
          
          if (renderHeight > pageHeight) {
            renderHeight = pageHeight;
            renderWidth = pageHeight * imgRatio;
          }
          
          // Center the image
          const x = (pageWidth - renderWidth) / 2;
          const y = (pageHeight - renderHeight) / 2;
          
          pdf.addImage(
            imgData.url, 
            'JPEG', 
            x, 
            y, 
            renderWidth, 
            renderHeight,
            undefined,
            'FAST'
          );
        }
        
        pdf.save('converted-images.pdf');
      } catch (error) {
        console.error('Error generating PDF:', error);
        alert('Failed to generate PDF');
      } finally {
        this.generating = false;
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
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 15px;
  margin-bottom: 20px;
}

.image-card {
  border: 1px solid #ddd;
  border-radius: 4px;
  overflow: hidden;
  background: #fff;
}

.image-card img {
  width: 100%;
  height: 150px;
  object-fit: cover;
  display: block;
}

.image-info {
  padding: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.image-info span {
  font-size: 12px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 80px;
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

.image-info button {
  background-color: #ff5252;
  padding: 4px 8px;
  font-size: 12px;
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

@media (max-width: 768px) {
  .upload-section {
    padding: 20px;
  }
}
</style>
