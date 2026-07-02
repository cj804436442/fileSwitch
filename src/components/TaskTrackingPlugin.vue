<template>
  <div class="task-tracking">
    <button
      v-if="showFloatingButton"
      ref="toggleBtn"
      class="task-tracking__button"
      type="button"
      @click="togglePanel"
    >
      <svg class="task-tracking__button-icon" viewBox="0 0 24 24" fill="#2563eb">
        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8l-6-6z" />
        <path d="M14 2v6h6" />
      </svg>
      <div class="task-tracking__button-text">
        <div>任务</div>
        <div>追踪</div>
      </div>
    </button>

    <div v-if="isPanelOpen" ref="panel" class="task-tracking__panel">
      <div class="task-tracking__header">
        <span>任务追踪</span>
        <button class="task-tracking__close" type="button" @click="closePanel">×</button>
      </div>

      <div class="task-tracking__main">
        <div class="task-tracking__sidebar">
          <button
            class="task-tracking__tab"
            :class="{ active: currentTab === 'message' }"
            type="button"
            @click="switchTab('message')"
          >消息</button>
          <button
            class="task-tracking__tab"
            :class="{ active: currentTab === 'task' }"
            type="button"
            @click="switchTab('task')"
          >任务</button>
        </div>

        <div class="task-tracking__content">
          <div v-if="loading" class="content-scroll">
            <div class="loading-wrap">
              <div class="spinner"></div>
              <span>加载中...</span>
            </div>
          </div>

          <div v-else-if="loadError" class="content-scroll">
            <div class="error-wrap">
              <span>⚠️ {{ loadError }}</span>
              <button class="error-retry" type="button" @click="fetchData">重新加载</button>
            </div>
          </div>

          <template v-else-if="currentDetailTask">
            <div class="content-scroll">
              <div class="detail-header">
                <button class="detail-back" type="button" @click="goBackToList">← 返回</button>
                <span class="detail-title">{{ currentDetailTask.name || '任务详情' }} › 任务跟踪</span>
              </div>

              <div v-if="currentDetailTask.loading" class="loading-wrap">
                <div class="spinner"></div>
                <span>加载任务进展...</span>
              </div>

              <div v-else-if="currentDetailTask.error" class="error-wrap">
                <span>⚠️ {{ currentDetailTask.error }}</span>
              </div>

              <div v-else-if="timelineEntries.length === 0" class="timeline-empty">暂无任务进展记录</div>

              <div v-else class="timeline-v2">
                <div v-for="entry in timelineEntries" :key="entry.key" class="tl-row">
                  <div class="tl-time">{{ entry.displayTime }}</div>

                  <div class="tl-dot-wrap">
                    <div class="tl-dot"></div>
                    <div class="tl-line"></div>
                  </div>

                  <div class="tl-card" :style="{ background: entry.cardBg }">
                    <div class="tl-card-header">
                      <span class="tl-user">{{ entry.submitterName || '系统' }}</span>

                      <div class="tl-card-actions">
                        <span
                          class="tl-tag"
                          :style="{
                            background: entry.typeColor + '20',
                            color: entry.typeColor,
                            border: '1px solid ' + entry.typeColor + '40'
                          }"
                        >{{ entry.typeName }}</span>

                        <button
                          class="tl-copy-btn"
                          type="button"
                          @click="copyContent(entry.content, entry.key)"
                        >
                          <svg
                            width="12"
                            height="12"
                            viewBox="0 0 24 24"
                            fill="none"
                            stroke="currentColor"
                            stroke-width="2"
                          >
                            <rect x="9" y="9" width="13" height="13" rx="2" />
                            <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
                          </svg>
                          {{ copiedTrackKey === entry.key ? '已复制' : '复制' }}
                        </button>
                      </div>
                    </div>

                    <div v-if="!entry.isCreate" class="tl-progress-row">
                      <span class="tl-progress-label">任务进度：</span>
                      <div class="tl-progress-bar-wrap">
                        <div class="tl-progress-bar" :style="{ width: entry.percent + '%' }"></div>
                      </div>
                      <span class="tl-progress-num">{{ entry.percent }}%</span>
                    </div>

                    <div
                      v-if="!entry.isCreate && entry.trackDate"
                      class="tl-meta-row"
                    >跟进日期：{{ entry.trackDate }}</div>

                    <div
                      class="tl-desc"
                    >{{ entry.isCreate ? '创建了任务 "' + entry.content + '"' : '内容描述：' + entry.content }}</div>

                    <div v-if="entry.attachments.length" class="tl-meta-row">
                      内容附件：
                      <button
                        v-for="(file, fileIndex) in entry.attachments"
                        :key="entry.key + '-file-' + fileIndex"
                        class="tl-attach-btn"
                        type="button"
                        @click="downloadAttachment(file)"
                      >📎 {{ file.originalName || '附件' }}</button>
                    </div>

                    <div v-if="entry.hyperlink" class="tl-meta-row">
                      超链接：
                      <a
                        class="tl-attach-link"
                        :href="entry.hyperlink"
                        target="_blank"
                        rel="noopener noreferrer"
                      >{{ entry.hyperlink }}</a>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </template>

          <template v-else-if="currentTab === 'message'">
            <div class="content-scroll">
              <div v-if="messages.length === 0" class="empty-state">暂无消息</div>

              <template v-else>
                <div
                  v-for="(message, index) in messages"
                  :key="message.id || 'msg-' + index"
                  class="message-item"
                  :class="{ clickable: isTaskMessage(message), external: !isTaskMessage(message) }"
                  @click="handleMessageClick(message)"
                >
                  <div class="message-icon">📩</div>
                  <div class="message-body">
                    <div>
                      <span class="message-sender">{{ message.sendUserName || '-' }}</span>
                      <span class="message-label">{{ message.type || '-' }}</span>
                    </div>
                    <div class="message-text">{{ message.content || '-' }}</div>
                    <div class="message-time">{{ message.createdAt || '-' }}</div>
                  </div>
                </div>
              </template>
            </div>

            <div class="load-more-bar">
              <button class="load-more-btn" type="button" @click="openExternal(messagePageUrl)">
                <svg
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                >
                  <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6" />
                  <polyline points="15 3 21 3 21 9" />
                  <line x1="10" y1="14" x2="21" y2="3" />
                </svg>
                查看全部消息
              </button>
            </div>
          </template>

          <template v-else>
            <div class="content-scroll">
              <div v-if="taskRows.length === 0" class="empty-state">暂无任务</div>

              <table v-else class="task-table">
                <thead>
                  <tr>
                    <th>任务名称</th>
                    <th>任务日志</th>
                    <th>状态</th>
                    <th>优先级</th>
                    <th>任务执行人</th>
                    <th>开始时间</th>
                    <th>截止时间</th>
                    <th>操作</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="task in taskRows" :key="task.id || task.taskName" class="task-row">
                    <td>{{ task.taskName || '-' }}</td>
                    <td>{{ task.taskLog || '-' }}</td>
                    <td>
                      <span
                        class="status-pill"
                        :style="{ color: task.statusMeta.textColor, background: task.statusMeta.bgColor }"
                      >{{ task.statusMeta.name }}</span>
                    </td>
                    <td>
                      <span
                        class="priority-pill"
                        :style="{ color: task.priorityMeta.textColor, background: task.priorityMeta.bgColor }"
                      >{{ task.priorityMeta.name }}</span>
                    </td>
                    <td>
                      <span
                        v-for="(name, executorIndex) in task.executors"
                        :key="task.id + '-executor-' + executorIndex"
                        class="executor-tag"
                      >{{ name }}</span>
                      <span v-if="task.executors.length === 0">-</span>
                    </td>
                    <td>{{ formatDate(task.startTime) }}</td>
                    <td>{{ formatDate(task.deadlineTime) }}</td>
                    <td>
                      <button
                        class="task-view-btn"
                        type="button"
                        @click="openTaskDetail(task.id, task.taskName)"
                      >查看</button>
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>

            <div class="load-more-bar">
              <button class="load-more-btn" type="button" @click="openExternal(taskPageUrl)">
                <svg
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                >
                  <path d="M18 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V8a2 2 0 0 1 2-2h6" />
                  <polyline points="15 3 21 3 21 9" />
                  <line x1="10" y1="14" x2="21" y2="3" />
                </svg>
                查看全部任务
              </button>
            </div>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
const STATUS_MAP = {
  NOT_STARTED: {
    name: "未开始",
    textColor: "#FFFFFF",
    bgColor: "#939CAC"
  },
  IN_PROGRESS: {
    name: "进行中",
    textColor: "#FFFFFF",
    bgColor: "#5393F3"
  },
  COMPLETED: {
    name: "已完成",
    textColor: "#FFFFFF",
    bgColor: "#00B69B"
  },
  TERMINATED: {
    name: "已终止",
    textColor: "#FFFFFF",
    bgColor: "#E88214"
  },
  BLOCKED: {
    name: "已停滞",
    textColor: "#FFFFFF",
    bgColor: "#F06868"
  },
  STALLED: {
    name: "已停滞",
    textColor: "#FFFFFF",
    bgColor: "#F06868"
  }
};

const PRIORITY_MAP = {
  URGENT_IMPORTANT: {
    name: "重要紧急",
    textColor: "#EE3321",
    bgColor: "#FDE6E4"
  },
  NOT_URGENT_IMPORTANT: {
    name: "重要非紧急",
    textColor: "#EFE9FE",
    bgColor: "#804DF6"
  },
  IMPORTANT_NOT_URGENT: {
    name: "重要非紧急",
    textColor: "#EFE9FE",
    bgColor: "#804DF6"
  },
  URGENT_NOT_IMPORTANT: {
    name: "紧急非重要",
    textColor: "#DF7400",
    bgColor: "#FAEBDB"
  },
  NOT_URGENT_NOT_IMPORTANT: {
    name: "非重要非紧急",
    textColor: "#E5F4F2",
    bgColor: "#00937D"
  },
  NOT_IMPORTANT_NOT_URGENT: {
    name: "非重要非紧急",
    textColor: "#E5F4F2",
    bgColor: "#00937D"
  }
};

const TRACK_TYPE_MAP = {
  "0": "任务创建",
  "1": "项目周报",
  "2": "进度更新",
  "3": "问题"
};

const TRACK_TYPE_COLOR_MAP = {
  "0": "#22c55e",
  "1": "#fb923c",
  "2": "#60a5fa",
  "3": "#f87171"
};

const TRACK_CARD_BG_MAP = {
  "0": "#f3fff7",
  "1": "#fefbed",
  "2": "#ffffff",
  "3": "#fbf7f3"
};

export default {
  name: "TaskTrackingPlugin",
  props: {
    messageApiUrl: {
      type: String,
      default: "/three/api/notice/queryByPage"
    },
    taskApiUrl: {
      type: String,
      default: "/three/api/dashboard/tasks"
    },
    taskDetailApiUrl: {
      type: String,
      default: "/three/api/task-track/list"
    },
    fileBaseUrl: {
      type: String,
      default: "http://2.50.54.219:8091"
    },
    messagePageUrl: {
      type: String,
      default: "http://2.50.54.219:8091/threeList/message"
    },
    taskPageUrl: {
      type: String,
      default: "http://2.50.54.219:8091/threeList/task/personal"
    },
    headers: {
      type: Object,
      default: () => ({
        "Content-Type": "application/json"
      })
    },
    showFloatingButton: {
      type: Boolean,
      default: true
    }
  },
  data() {
    return {
      messages: [],
      tasks: [],
      currentTab: "message",
      currentDetailTask: null,
      isPanelOpen: false,
      loading: false,
      loadError: "",
      downloading: false,
      copiedTrackKey: "",
      copyResetTimer: null
    };
  },
  computed: {
    taskRows() {
      return this.tasks.map(task => ({
        ...task,
        statusMeta: this.getStatusMeta(task),
        priorityMeta: this.getPriorityMeta(task),
        executors: this.parseExecutors(task.executorRealNames)
      }));
    },
    timelineEntries() {
      if (
        !this.currentDetailTask ||
        !Array.isArray(this.currentDetailTask.progress)
      ) {
        return [];
      }

      return this.currentDetailTask.progress.map((item, index) => {
        const typeKey = String(item.trackType ?? "");
        const timeText = item.submitTime || item.createTime || "";
        const parsedPercent = Number(item.taskProgress ?? item.percent ?? 0);
        const percent = Number.isFinite(parsedPercent)
          ? Math.max(0, Math.min(100, parsedPercent))
          : 0;

        return {
          key: `${item.id || "track"}-${index}-${timeText}`,
          isCreate: typeKey === "0",
          typeName: TRACK_TYPE_MAP[typeKey] || "进展",
          typeColor: TRACK_TYPE_COLOR_MAP[typeKey] || "#93c5fd",
          cardBg: TRACK_CARD_BG_MAP[typeKey] || "#ffffff",
          displayTime: this.formatDateTime(timeText),
          percent,
          trackDate: item.trackDate || "",
          submitterName: item.submitterName || "",
          content: item.content || "",
          hyperlink: item.hyperlink || "",
          attachments: this.parseAttachments(item.attachmentPath)
        };
      });
    }
  },
  mounted() {
    document.addEventListener("click", this.handleDocumentClick, true);
  },
  beforeDestroy() {
    document.removeEventListener("click", this.handleDocumentClick, true);
    if (this.copyResetTimer) {
      clearTimeout(this.copyResetTimer);
      this.copyResetTimer = null;
    }
  },
  methods: {
    getTokenFromCookie() {
      const match = document.cookie
        .split(";")
        .map(cookie => cookie.trim())
        .find(cookie => cookie.startsWith("Admin-Token-Portal="));
      return match
        ? decodeURIComponent(match.slice(match.indexOf("=") + 1))
        : "";
    },
    buildHeaders(contentType = false) {
      const token = this.getTokenFromCookie();
      const merged = {
        ...(contentType ? { "Content-Type": "application/json" } : {}),
        ...this.headers
      };

      if (token) {
        merged["Admin-Token-Portal"] = token;
      }

      return merged;
    },
    togglePanel() {
      if (this.isPanelOpen) {
        this.closePanel();
      } else {
        this.openPanel();
      }
    },
    openPanel() {
      if (this.isPanelOpen) {
        return;
      }
      this.isPanelOpen = true;
      this.currentDetailTask = null;
      this.fetchData();
    },
    closePanel() {
      this.isPanelOpen = false;
      this.currentDetailTask = null;
    },
    handleDocumentClick(event) {
      if (!this.isPanelOpen || this.downloading) {
        return;
      }

      const path =
        typeof event.composedPath === "function" ? event.composedPath() : [];
      const clickedInside = path.length
        ? path.includes(this.$el)
        : this.$el.contains(event.target);

      if (!clickedInside) {
        this.closePanel();
      }
    },
    switchTab(tab) {
      this.currentTab = tab;
      this.currentDetailTask = null;
    },
    async fetchData() {
      this.loading = true;
      this.loadError = "";

      try {
        const [messages, tasks] = await Promise.all([
          this.fetchMessages(),
          this.fetchTasks()
        ]);
        this.messages = messages;
        this.tasks = tasks;
      } catch (error) {
        this.loadError = "数据加载失败，请重试";
      } finally {
        this.loading = false;
      }
    },
    async fetchMessages() {
      const response = await fetch(this.messageApiUrl, {
        method: "POST",
        headers: this.buildHeaders(true),
        body: JSON.stringify({ pageSize: 20, pageNum: 1 })
      });

      if (!response.ok) {
        throw new Error(`消息接口异常: ${response.status}`);
      }

      const json = await response.json();
      return json && json.data && Array.isArray(json.data.records)
        ? json.data.records
        : [];
    },
    async fetchTasks() {
      const response = await fetch(this.taskApiUrl, {
        method: "POST",
        headers: this.buildHeaders(true),
        body: JSON.stringify({ pageSize: 20, pageNum: 1 })
      });

      if (!response.ok) {
        throw new Error(`任务接口异常: ${response.status}`);
      }

      const json = await response.json();
      return json && json.data && Array.isArray(json.data.list)
        ? json.data.list
        : [];
    },
    async fetchTaskDetail(taskId) {
      const base = String(this.taskDetailApiUrl || "/api/tasks").replace(
        /\/+$/,
        ""
      );
      const response = await fetch(`${base}/${taskId}`, {
        method: "GET",
        headers: this.buildHeaders(true)
      });

      if (!response.ok) {
        throw new Error(`任务详情接口异常: ${response.status}`);
      }

      const json = await response.json();
      return json && Array.isArray(json.data) ? json.data : [];
    },
    isTaskMessage(message) {
      return message && message.type === "TASK";
    },
    handleMessageClick(message) {
      if (!this.isTaskMessage(message) || !message.taskId) {
        return;
      }
      this.openTaskDetail(message.taskId, message.sendUserName || "");
    },
    async openTaskDetail(taskId, taskName) {
      if (!taskId) {
        return;
      }

      this.currentDetailTask = {
        id: taskId,
        name: taskName,
        progress: [],
        loading: true,
        error: ""
      };

      try {
        const progress = await this.fetchTaskDetail(taskId);
        this.currentDetailTask = {
          id: taskId,
          name: taskName,
          progress,
          loading: false,
          error: ""
        };
      } catch (error) {
        this.currentDetailTask = {
          id: taskId,
          name: taskName,
          progress: [],
          loading: false,
          error: "任务进展加载失败"
        };
      }
    },
    goBackToList() {
      this.currentDetailTask = null;
    },
    openExternal(url) {
      if (!url) {
        return;
      }
      window.open(url, "_blank", "noopener,noreferrer");
    },
    parseExecutors(names) {
      if (!names) {
        return [];
      }
      return String(names)
        .split(",")
        .map(name => name.trim())
        .filter(Boolean);
    },
    getStatusMeta(task) {
      const fallback = {
        name: task.taskStatusName || "-",
        textColor: "#ffffff",
        bgColor: "#939CAC"
      };
      return STATUS_MAP[task.taskStatus] || fallback;
    },
    getPriorityMeta(task) {
      const fallback = {
        name: task.priorityName || "-",
        textColor: "#333333",
        bgColor: "#eeeeee"
      };
      return PRIORITY_MAP[task.priority] || fallback;
    },
    parseAttachments(rawAttachmentPath) {
      if (!rawAttachmentPath) {
        return [];
      }

      if (Array.isArray(rawAttachmentPath)) {
        return rawAttachmentPath;
      }

      if (typeof rawAttachmentPath === "string") {
        try {
          const parsed = JSON.parse(rawAttachmentPath);
          return Array.isArray(parsed) ? parsed : [];
        } catch (error) {
          return [];
        }
      }

      return [];
    },
    formatDate(value) {
      if (!value) {
        return "";
      }
      return String(value).slice(0, 10);
    },
    formatDateTime(value) {
      if (!value) {
        return "";
      }
      return String(value)
        .slice(0, 16)
        .replace("T", " ");
    },
    async copyContent(content, key) {
      try {
        if (navigator.clipboard && navigator.clipboard.writeText) {
          await navigator.clipboard.writeText(content || "");
        } else {
          const textarea = document.createElement("textarea");
          textarea.value = content || "";
          textarea.style.position = "fixed";
          textarea.style.opacity = "0";
          document.body.appendChild(textarea);
          textarea.select();
          document.execCommand("copy");
          document.body.removeChild(textarea);
        }

        this.copiedTrackKey = key;
        if (this.copyResetTimer) {
          clearTimeout(this.copyResetTimer);
        }
        this.copyResetTimer = setTimeout(() => {
          this.copiedTrackKey = "";
        }, 1500);
      } catch (error) {
        alert("复制失败，请手动复制");
      }
    },
    async downloadAttachment(file = {}) {
      this.downloading = true;

      try {
        const payload = {
          fileId: file.fileId || null,
          filePath: file.filePath || null,
          fileSize: file.fileSize ? Number(file.fileSize) : null,
          fileStatus: file.fileStatus || null,
          fileSuffix: file.fileSuffix || null,
          originalName: file.originalName || null,
          storedName: file.storedName || null,
          uploadTime: file.uploadTime || null,
          userCode: file.userCode || null
        };

        const response = await fetch("/three/api/file/downLoadFile", {
          method: "POST",
          headers: this.buildHeaders(true),
          body: JSON.stringify(payload)
        });

        if (!response.ok) {
          throw new Error("下载失败");
        }

        const blob = await response.blob();
        const disposition = response.headers.get("Content-Disposition");
        let fileName = file.originalName || "file";

        if (disposition) {
          const match =
            disposition.match(/filename\*=UTF-8''(.+)/) ||
            disposition.match(/filename="?([^"]+)"?/);
          if (match) {
            fileName = decodeURIComponent(match[1]);
          }
        }

        const blobUrl = URL.createObjectURL(blob);
        const link = document.createElement("a");
        link.href = blobUrl;
        link.download = fileName;
        link.dispatchEvent(new MouseEvent("click", { bubbles: false }));
        URL.revokeObjectURL(blobUrl);
      } catch (error) {
        alert("文件下载失败，请重试");
      } finally {
        this.downloading = false;
      }
    }
  }
};
</script>

<style scoped>
.task-tracking * {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  font-family: system-ui, -apple-system, sans-serif;
}

.task-tracking__button {
  position: fixed;
  right: 24px;
  bottom: 24px;
  width: 68px;
  height: 112px;
  background: #ffffff;
  border-radius: 34px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.18);
  border: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  cursor: pointer;
  transition: all 0.25s ease;
  z-index: 100000;
  user-select: none;
}

.task-tracking__button:hover {
  transform: translateY(-3px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.22);
}

.task-tracking__button-icon {
  width: 32px;
  height: 32px;
}

.task-tracking__button-text {
  text-align: center;
  color: #222222;
  font-size: 11px;
  line-height: 1.25;
  font-weight: 600;
}

.task-tracking__panel {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 75vw;
  height: 60vh;
  min-width: 800px;
  min-height: 500px;
  max-width: 1400px;
  max-height: 90vh;
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  z-index: 100001;
  overflow: hidden;
  border: 1px solid #dddddd;
}

.task-tracking__header {
  padding: 16px 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #eeeeee;
  font-size: 18px;
  font-weight: 600;
  background: #fafafa;
}

.task-tracking__close {
  cursor: pointer;
  font-size: 24px;
  color: #888888;
  border: none;
  background: transparent;
  line-height: 1;
}

.task-tracking__main {
  display: flex;
  flex: 1;
  overflow: hidden;
}

.task-tracking__sidebar {
  width: 100px;
  background: #f8f9fa;
  border-right: 1px solid #eeeeee;
  padding: 20px 0;
  flex-shrink: 0;
}

.task-tracking__tab {
  width: 100%;
  border: none;
  background: transparent;
  padding: 16px 12px;
  text-align: center;
  font-size: 15px;
  cursor: pointer;
  transition: all 0.2s;
}

.task-tracking__tab.active {
  background: #e6f0ff;
  color: #2563eb;
  font-weight: 600;
  border-right: 3px solid #2563eb;
}

.task-tracking__content {
  flex: 1;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.content-scroll {
  flex: 1;
  padding: 20px;
  overflow: auto;
}

.load-more-bar {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 14px 20px;
  border-top: 1px solid #f0f0f0;
  background: #fafafa;
}

.load-more-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 7px 20px;
  font-size: 13px;
  color: #2563eb;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 999px;
  cursor: pointer;
  transition: all 0.2s;
  font-family: system-ui, -apple-system, sans-serif;
}

.load-more-btn:hover {
  background: #dbeafe;
  border-color: #93c5fd;
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.15);
}

.load-more-btn svg {
  width: 14px;
  height: 14px;
  flex-shrink: 0;
}

.loading-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 200px;
  color: #888888;
  font-size: 15px;
  gap: 10px;
}

.spinner {
  width: 22px;
  height: 22px;
  border: 3px solid #e5e7eb;
  border-top-color: #2563eb;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.empty-state {
  text-align: center;
  padding: 60px;
  color: #888888;
}

.message-item {
  display: flex;
  padding: 14px 16px;
  margin-bottom: 12px;
  border-radius: 8px;
  background: white;
  border: 1px solid #eeeeee;
  transition: all 0.2s;
}

.message-item.clickable {
  cursor: pointer;
}

.message-item.clickable:hover {
  background: #f0f7ff;
  border-color: #bfdbfe;
}

.message-item.external {
  cursor: default;
  opacity: 0.85;
}

.message-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  margin-right: 16px;
  flex-shrink: 0;
  background: #f0f0f0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
}

.message-body {
  flex: 1;
}

.message-sender {
  font-weight: 600;
  margin-bottom: 4px;
}

.message-label {
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 12px;
  margin-left: 8px;
  background: #e0f2fe;
  color: #0369a1;
}

.message-text {
  color: #444444;
  line-height: 1.5;
  margin: 6px 0;
}

.message-time {
  color: #888888;
  font-size: 12px;
}

.task-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

.task-table th {
  background: #f8f9fa;
  padding: 12px 16px;
  text-align: left;
  font-weight: 500;
  color: #555555;
  border-bottom: 2px solid #dddddd;
  white-space: nowrap;
}

.task-table td {
  padding: 12px 16px;
  border-bottom: 1px solid #eeeeee;
  vertical-align: middle;
}

.task-table tr:hover {
  background: #f9fcff;
}

.status-pill,
.priority-pill {
  padding: 4px 10px;
  border-radius: 999px;
  font-size: 12px;
  font-weight: 500;
  white-space: nowrap;
  display: inline-block;
}

.executor-tag {
  display: inline-block;
  padding: 2px 8px;
  background: #f1f5f9;
  border-radius: 4px;
  font-size: 12px;
  color: #475569;
  margin: 2px;
}

.task-view-btn {
  background: none;
  border: none;
  color: #2563eb;
  font-size: 13px;
  cursor: pointer;
  padding: 0;
  font-family: system-ui, -apple-system, sans-serif;
}

.task-view-btn:hover {
  text-decoration: underline;
}

.detail-header {
  padding: 16px 24px;
  border-bottom: 1px solid #eeeeee;
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 15px;
  color: #555555;
  background: #fafafa;
  margin: -20px -20px 0;
}

.detail-back {
  color: #2563eb;
  cursor: pointer;
  font-weight: 500;
  flex-shrink: 0;
  border: none;
  background: transparent;
}

.detail-title {
  font-weight: 600;
  color: #1e293b;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.error-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 200px;
  color: #ef4444;
  gap: 12px;
}

.error-retry {
  padding: 6px 18px;
  border-radius: 6px;
  border: 1px solid #ef4444;
  background: white;
  color: #ef4444;
  cursor: pointer;
  font-size: 14px;
}

.error-retry:hover {
  background: #fef2f2;
}

.timeline-v2 {
  padding: 24px 20px;
}

.tl-row {
  display: flex;
  margin-bottom: 8px;
}

.tl-time {
  width: 140px;
  flex-shrink: 0;
  font-size: 13px;
  color: #94a3b8;
  padding-top: 4px;
  text-align: right;
  padding-right: 16px;
}

.tl-dot-wrap {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex-shrink: 0;
  width: 20px;
}

.tl-dot {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: white;
  border: 3px solid #60a5fa;
  flex-shrink: 0;
  margin-top: 4px;
  z-index: 1;
}

.tl-line {
  width: 2px;
  flex: 1;
  min-height: 40px;
  background: #e2e8f0;
  margin-top: 4px;
}

.tl-row:last-child .tl-line {
  display: none;
}

.tl-card {
  flex: 1;
  margin-left: 16px;
  margin-bottom: 16px;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 16px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.05);
}

.tl-card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.tl-user {
  font-weight: 600;
  font-size: 15px;
  color: #1e293b;
}

.tl-card-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.tl-tag {
  font-size: 12px;
  padding: 3px 10px;
  border-radius: 999px;
  font-weight: 500;
}

.tl-copy-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: #60a5fa;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 999px;
  padding: 3px 10px;
  cursor: pointer;
  font-family: system-ui, -apple-system, sans-serif;
}

.tl-copy-btn:hover {
  background: #dbeafe;
}

.tl-progress-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
  width: 100%;
}

.tl-progress-label {
  font-size: 13px;
  color: #64748b;
  white-space: nowrap;
}

.tl-progress-bar-wrap {
  flex: 1;
  height: 8px;
  min-width: 0;
  background: #e2e8f0;
  border-radius: 999px;
  overflow: hidden;
}

.tl-progress-bar {
  height: 100%;
  background: linear-gradient(90deg, #60a5fa, #93c5fd);
  border-radius: 999px;
  transition: width 0.4s;
}

.tl-progress-num {
  font-size: 13px;
  color: #475569;
  white-space: nowrap;
  min-width: 36px;
}

.tl-meta-row {
  font-size: 13px;
  color: #64748b;
  margin-bottom: 6px;
}

.tl-desc {
  font-size: 14px;
  color: #334155;
  line-height: 1.6;
  margin-bottom: 6px;
}

.tl-attach-btn {
  background: none;
  border: none;
  color: #2563eb;
  font-size: 13px;
  cursor: pointer;
  padding: 0;
  font-family: system-ui, -apple-system, sans-serif;
  text-decoration: underline;
  margin-right: 8px;
}

.tl-attach-btn:hover {
  color: #1d4ed8;
}

.tl-attach-link {
  color: #2563eb;
  text-decoration: none;
}

.tl-attach-link:hover {
  text-decoration: underline;
}

.timeline-empty {
  text-align: center;
  padding: 60px;
  color: #888888;
}

@media (max-width: 960px) {
  .task-tracking__panel {
    width: calc(100vw - 24px);
    height: calc(100vh - 36px);
    min-width: 0;
    min-height: 0;
    border-radius: 10px;
  }

  .task-tracking__sidebar {
    width: 78px;
  }

  .task-table {
    font-size: 12px;
  }

  .task-table th,
  .task-table td {
    padding: 10px 8px;
  }

  .tl-time {
    width: 92px;
    font-size: 12px;
  }
}
</style>
