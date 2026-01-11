<template>
  <div class="app-layout">
    <!-- Mobile Header -->
    <header class="mobile-header" v-if="isMobile">
      <button class="menu-btn" @click="toggleSidebar">
        <svg viewBox="0 0 24 24" width="24" height="24">
          <path fill="currentColor" d="M3 18h18v-2H3v2zm0-5h18v-2H3v2zm0-7v2h18V6H3z"/>
        </svg>
      </button>
      <h1>文件转换</h1>
    </header>

    <!-- Sidebar Overlay -->
    <div 
      class="sidebar-overlay" 
      v-if="isMobile && sidebarOpen"
      @click="closeSidebar"
    ></div>

    <aside class="sidebar" :class="{ 'open': sidebarOpen, 'mobile': isMobile }">
      <div class="brand">
        <h1>文件转换</h1>
      </div>
      <nav class="nav-menu">
        <button 
          v-for="tab in tabs" 
          :key="tab.id"
          :class="{ active: currentTab === tab.id }"
          @click="selectTab(tab.id)"
          class="nav-item"
        >
          {{ tab.label }}
        </button>
      </nav>
      <div class="sidebar-footer">
        <p>v1.0.0</p>
      </div>
    </aside>

    <main class="main-content">
      <header class="top-bar" v-if="!isMobile">
        <h2>{{ currentTabLabel }}</h2>
      </header>
      <div class="content-scroll">
        <keep-alive>
          <component :is="currentTabComponent" />
        </keep-alive>
      </div>
    </main>
  </div>
</template>

<script>
import ImageToPdf from './components/ImageToPdf.vue';
import PdfToImage from './components/PdfToImage.vue';
import WordToPdf from './components/WordToPdf.vue';
import PdfToWord from './components/PdfToWord.vue';
import PdfToPpt from './components/PdfToPpt.vue';
import PptToPdf from './components/PptToPdf.vue';
import PdfMerge from './components/PdfMerge.vue';

export default {
  name: 'App',
  components: {
    ImageToPdf,
    PdfToImage,
    WordToPdf,
    PdfToWord,
    PdfToPpt,
    PptToPdf,
    PdfMerge
  },
  data() {
    return {
      currentTab: 'img2pdf',
      sidebarOpen: false,
      isMobile: false,
      tabs: [
        { id: 'img2pdf', label: '图片转 PDF', component: 'ImageToPdf' },
        { id: 'pdf2img', label: 'PDF 转图片', component: 'PdfToImage' },
        { id: 'word2pdf', label: 'Word 转 PDF', component: 'WordToPdf' },
        { id: 'pdf2word', label: 'PDF 转 Word', component: 'PdfToWord' },
        { id: 'pdf2ppt', label: 'PDF 转 PPT', component: 'PdfToPpt' },
        { id: 'ppt2pdf', label: 'PPT 转 PDF', component: 'PptToPdf' },
        { id: 'pdfmerge', label: 'PDF 合并', component: 'PdfMerge' }
      ]
    };
  },
  computed: {
    currentTabComponent() {
      const tab = this.tabs.find(t => t.id === this.currentTab);
      return tab ? tab.component : 'ImageToPdf';
    },
    currentTabLabel() {
      const tab = this.tabs.find(t => t.id === this.currentTab);
      return tab ? tab.label : 'File Converter';
    }
  },
  mounted() {
    this.checkMobile();
    window.addEventListener('resize', this.checkMobile);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.checkMobile);
  },
  methods: {
    checkMobile() {
      this.isMobile = window.innerWidth <= 768;
      if (!this.isMobile) {
        this.sidebarOpen = false;
      }
    },
    toggleSidebar() {
      this.sidebarOpen = !this.sidebarOpen;
    },
    closeSidebar() {
      this.sidebarOpen = false;
    },
    selectTab(id) {
      this.currentTab = id;
      if (this.isMobile) {
        this.closeSidebar();
      }
    }
  }
};
</script>

<style>
/* Reset & Global */
body {
  margin: 0;
  padding: 0;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background-color: #f5f7fa;
  color: #333;
}

* {
  box-sizing: border-box;
}

/* Layout */
.app-layout {
  display: flex;
  height: 100vh;
  overflow: hidden;
  position: relative;
}

/* Mobile Header */
.mobile-header {
  display: none;
  background-color: #2c3e50;
  color: white;
  padding: 10px 15px;
  align-items: center;
  width: 100%;
  height: 60px;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 100;
}

.mobile-header h1 {
  margin: 0;
  font-size: 18px;
  margin-left: 15px;
  font-weight: 500;
}

.menu-btn {
  background: none;
  border: none;
  color: white;
  cursor: pointer;
  padding: 5px;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Sidebar */
.sidebar {
  width: 260px;
  background-color: #2c3e50;
  color: #ecf0f1;
  display: flex;
  flex-direction: column;
  box-shadow: 2px 0 5px rgba(0,0,0,0.1);
  flex-shrink: 0;
  transition: transform 0.3s ease;
  z-index: 1001;
}

.sidebar.mobile {
  position: fixed;
  top: 0;
  left: 0;
  height: 100%;
  transform: translateX(-100%);
}

.sidebar.mobile.open {
  transform: translateX(0);
}

.sidebar-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 1000;
}

.brand {
  padding: 20px;
  background-color: #1a252f;
  text-align: center;
}

.brand h1 {
  margin: 0;
  font-size: 24px;
  font-weight: 600;
  letter-spacing: 1px;
}

.nav-menu {
  flex: 1;
  padding: 20px 0;
  overflow-y: auto;
}

.nav-item {
  display: block;
  width: 100%;
  padding: 15px 25px;
  border: none;
  background: none;
  color: #bdc3c7;
  text-align: left;
  cursor: pointer;
  font-size: 16px;
  transition: all 0.2s ease;
  border-left: 4px solid transparent;
}

.nav-item:hover {
  background-color: #34495e;
  color: white;
}

.nav-item.active {
  background-color: #34495e;
  color: #fff;
  border-left-color: #4caf50;
  font-weight: 600;
}

.sidebar-footer {
  padding: 15px;
  text-align: center;
  font-size: 12px;
  color: #7f8c8d;
  background-color: #1a252f;
}

/* Main Content */
.main-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  background-color: #f5f7fa;
  min-width: 0;
  height: 100%;
}

.top-bar {
  background-color: white;
  padding: 15px 30px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
  display: flex;
  align-items: center;
}

.top-bar h2 {
  margin: 0;
  font-size: 20px;
  color: #2c3e50;
}

.content-scroll {
  flex: 1;
  overflow-y: auto;
  padding: 30px;
  -webkit-overflow-scrolling: touch;
}

/* Common Components Style Override */
.tool-container {
  background: white;
  border-radius: 8px;
  max-width: 100%;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
  padding: 30px;
  margin: 0 auto;
}

/* Responsive Styles */
@media (max-width: 768px) {
  .app-layout {
    flex-direction: column;
  }

  .mobile-header {
    display: flex;
  }

  .main-content {
    padding-top: 60px; /* Space for header */
  }

  .content-scroll {
    padding: 15px;
  }
  
  .brand {
    display: none; /* Hide sidebar brand on mobile since we have header */
  }
  
  .sidebar.mobile {
    padding-top: 0;
  }
  
  .tool-container {
    padding: 20px;
  }
}
</style>
