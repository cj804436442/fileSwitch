<template>
  <div class="tool-container location-page">
    <section class="location-hero">
      <div>
        <p class="eyebrow">AMap H5 Demo</p>
        <h2>高德定位</h2>
        <p class="hero-desc">
          用高德 JS API 获取手机 H5 当前定位，适合在 HTTPS 环境或 localhost 调试。
        </p>
      </div>
      <button class="locate-btn" :disabled="loading" @click="getCurrentLocation">
        {{ loading ? '定位中...' : '获取当前位置' }}
      </button>
    </section>

    <!-- <section class="config-panel">
      <label class="field">
        <span>高德 Web JS API Key</span>
        <input
          v-model.trim="runtimeKey"
          type="text"
          placeholder="优先读取 VITE_AMAP_KEY，也可以临时输入"
          autocomplete="off"
        />
      </label>
      <label class="field">
        <span>安全密钥 securityJsCode</span>
        <input
          v-model.trim="runtimeSecurityCode"
          type="text"
          placeholder="可选：优先读取 VITE_AMAP_SECURITY_JS_CODE"
          autocomplete="off"
        />
      </label>
      <button class="save-btn" @click="saveRuntimeConfig">保存到本地</button>
    </section> -->

    <section class="status-card" :class="statusType">
      <strong>{{ statusTitle }}</strong>
      <p>{{ statusMessage }}</p>
    </section>

    <section v-if="location" class="result-grid">
      <div class="result-item">
        <span>经度</span>
        <strong>{{ location.lng }}</strong>
      </div>
      <div class="result-item">
        <span>纬度</span>
        <strong>{{ location.lat }}</strong>
      </div>
      <div class="result-item">
        <span>精度</span>
        <strong>{{ location.accuracy || '-' }} 米</strong>
      </div>
      <div class="result-item">
        <span>定位类型</span>
        <strong>{{ location.locationType || '-' }}</strong>
      </div>
      <div class="result-item full">
        <span>地址</span>
        <strong>{{ location.address || '暂无地址信息' }}</strong>
      </div>
    </section>
  </div>
</template>

<script>
const AMAP_SCRIPT_ID = 'amap-jsapi-script';
const STORAGE_KEY = 'file-switch-amap-location-config';

function readEnvValue(key) {
  return import.meta.env && import.meta.env[key] ? import.meta.env[key] : '';
}

export default {
  name: 'GetLoaction',
  data() {
    const savedConfig = this.readSavedConfig();

    return {
      runtimeKey: readEnvValue('VITE_AMAP_KEY') || savedConfig.key || '4c84a617c282498b6acb43d5fb89a2ef',
      runtimeSecurityCode: readEnvValue('VITE_AMAP_SECURITY_JS_CODE') || savedConfig.securityCode || '54968ad8b52c8bc63634c4747711eeb7',
      loading: false,
      statusType: 'idle',
      statusTitle: '准备就绪',
      statusMessage: '填写高德 Key 后点击按钮，即可在手机 H5 中发起定位。',
      location: null
    };
  },
  methods: {
    readSavedConfig() {
      try {
        return JSON.parse(window.localStorage.getItem(STORAGE_KEY)) || {};
      } catch (error) {
        return {};
      }
    },
    saveRuntimeConfig() {
      window.localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify({
          key: this.runtimeKey,
          securityCode: this.runtimeSecurityCode
        })
      );
      this.setStatus('success', '已保存', 'Key 信息已保存到当前浏览器，方便本机 demo 调试。');
    },
    setStatus(type, title, message) {
      this.statusType = type;
      this.statusTitle = title;
      this.statusMessage = message;
    },
    loadAMap() {
      if (window.AMap && window.AMap.Geolocation) {
        return Promise.resolve(window.AMap);
      }

      if (!this.runtimeKey) {
        return Promise.reject(new Error('请先填写高德 Web JS API Key。'));
      }

      if (this.runtimeSecurityCode) {
        window._AMapSecurityConfig = {
          securityJsCode: this.runtimeSecurityCode
        };
      }

      const existingScript = document.getElementById(AMAP_SCRIPT_ID);
      if (existingScript) {
        if (existingScript.dataset.loaded === 'true' && window.AMap) {
          return Promise.resolve(window.AMap);
        }

        if (existingScript.dataset.error === 'true') {
          existingScript.remove();
          return this.loadAMap();
        }

        return new Promise((resolve, reject) => {
          existingScript.addEventListener('load', () => resolve(window.AMap), { once: true });
          existingScript.addEventListener('error', () => reject(new Error('高德 JS API 加载失败。')), { once: true });
        });
      }

      return new Promise((resolve, reject) => {
        const script = document.createElement('script');
        script.id = AMAP_SCRIPT_ID;
        script.src = `https://webapi.amap.com/maps?v=2.0&key=${encodeURIComponent(this.runtimeKey)}&plugin=AMap.Geolocation,AMap.Geocoder`;
        script.async = true;
        script.onload = () => {
          script.dataset.loaded = 'true';
          resolve(window.AMap);
        };
        script.onerror = () => {
          script.dataset.error = 'true';
          reject(new Error('高德 JS API 加载失败，请检查网络或 Key 配置。'));
        };
        document.head.appendChild(script);
      });
    },
    getAddressByLngLat(AMap, lng, lat, fallbackAddress) {
      if (fallbackAddress) {
        return Promise.resolve(fallbackAddress);
      }

      if (!lng || !lat) {
        return Promise.resolve('');
      }

      return new Promise((resolve) => {
        AMap.plugin('AMap.Geocoder', () => {
          const geocoder = new AMap.Geocoder();

          geocoder.getAddress([lng, lat], (status, result) => {
            if (status === 'complete' && result.regeocode) {
              resolve(result.regeocode.formattedAddress || '');
              return;
            }

            resolve('');
          });
        });
      });
    },
    getCurrentLocation() {
      this.loading = true;
      this.location = null;
      this.setStatus('loading', '正在定位', '正在唤起浏览器定位权限，请在手机上允许访问位置。');

      this.loadAMap()
        .then((AMap) => {
          const geolocation = new AMap.Geolocation({
            enableHighAccuracy: true,
            timeout: 10000,
            maximumAge: 0,
            convert: true,
            showButton: false,
            showMarker: false,
            showCircle: false,
            panToLocation: false,
            zoomToAccuracy: false,
            extensions: 'all'
          });

          geolocation.getCurrentPosition((status, result) => {
            if (status === 'complete') {
              const lng = result.position && result.position.lng;
              const lat = result.position && result.position.lat;

              this.location = {
                lng,
                lat,
                accuracy: result.accuracy,
                locationType: result.location_type,
                address: result.formattedAddress || '地址解析中...'
              };
              this.setStatus('loading', '定位成功', '已获取经纬度，正在解析详细地址。');

              this.getAddressByLngLat(AMap, lng, lat, result.formattedAddress)
                .then((address) => {
                  this.location.address = address || '暂无地址信息';
                  this.setStatus('success', '定位成功', '已获取当前经纬度和地址信息。');
                })
                .finally(() => {
                  this.loading = false;
                });
              return;
            }

            this.loading = false;
            const message = result && (result.message || result.info) ? `${result.info || ''} ${result.message || ''}`.trim() : '定位失败。';
            this.setStatus('error', '定位失败', message);
          });
        })
        .catch((error) => {
          this.loading = false;
          this.setStatus('error', '无法发起定位', error.message);
        });
    }
  }
};
</script>

<style scoped>
.location-page {
  max-width: 860px;
}

.location-hero {
  display: flex;
  justify-content: space-between;
  gap: 24px;
  align-items: center;
  padding-bottom: 24px;
  border-bottom: 1px solid #edf0f4;
}

.eyebrow {
  margin: 0 0 8px;
  color: #4caf50;
  font-size: 13px;
  font-weight: 700;
}

.location-hero h2 {
  margin: 0;
  color: #2c3e50;
  font-size: 28px;
}

.hero-desc {
  max-width: 540px;
  margin: 10px 0 0;
  color: #657080;
  line-height: 1.7;
}

.locate-btn,
.save-btn {
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 15px;
  font-weight: 600;
  transition: transform 0.2s ease, box-shadow 0.2s ease, background-color 0.2s ease;
}

.locate-btn {
  min-width: 150px;
  padding: 13px 18px;
  color: #fff;
  background: #2c3e50;
  box-shadow: 0 8px 18px rgba(44, 62, 80, 0.18);
}

.locate-btn:disabled {
  cursor: not-allowed;
  opacity: 0.7;
}

.save-btn {
  align-self: flex-end;
  padding: 11px 16px;
  color: #2c3e50;
  background: #eef4f0;
}

.locate-btn:not(:disabled):hover,
.save-btn:hover {
  transform: translateY(-1px);
}

.config-panel {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr)) auto;
  gap: 14px;
  margin-top: 24px;
  align-items: end;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 8px;
  color: #2c3e50;
  font-size: 14px;
  font-weight: 600;
}

.field input {
  width: 100%;
  min-height: 42px;
  padding: 10px 12px;
  border: 1px solid #dce2e8;
  border-radius: 6px;
  color: #2c3e50;
  font-size: 14px;
  outline: none;
}

.field input:focus {
  border-color: #4caf50;
  box-shadow: 0 0 0 3px rgba(76, 175, 80, 0.12);
}

.status-card {
  margin-top: 22px;
  padding: 16px;
  border-radius: 8px;
  border: 1px solid #dfe6ee;
  background: #f8fafc;
}

.status-card strong {
  display: block;
  color: #2c3e50;
  font-size: 16px;
}

.status-card p {
  margin: 8px 0 0;
  color: #657080;
  line-height: 1.6;
}

.status-card.success {
  border-color: #b8dfc0;
  background: #f2fbf4;
}

.status-card.error {
  border-color: #f0b9b9;
  background: #fff6f6;
}

.status-card.loading {
  border-color: #b8cee8;
  background: #f3f8ff;
}

.result-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
  margin-top: 22px;
}

.result-item {
  padding: 16px;
  border: 1px solid #edf0f4;
  border-radius: 8px;
  background: #fff;
}

.result-item.full {
  grid-column: 1 / -1;
}

.result-item span {
  display: block;
  margin-bottom: 8px;
  color: #8a95a3;
  font-size: 13px;
}

.result-item strong {
  display: block;
  color: #2c3e50;
  font-size: 18px;
  line-height: 1.5;
  word-break: break-all;
}

.tips {
  margin-top: 24px;
  padding-top: 20px;
  border-top: 1px solid #edf0f4;
}

.tips h3 {
  margin: 0 0 12px;
  color: #2c3e50;
  font-size: 18px;
}

.tips ul {
  margin: 0;
  padding-left: 20px;
  color: #657080;
  line-height: 1.8;
}

@media (max-width: 768px) {
  .location-hero,
  .config-panel {
    grid-template-columns: 1fr;
  }

  .location-hero {
    display: grid;
  }

  .locate-btn,
  .save-btn {
    width: 100%;
  }

  .result-grid {
    grid-template-columns: 1fr;
  }
}
</style>
