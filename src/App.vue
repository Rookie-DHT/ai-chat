<template>
  <div class="chat-container">
    <!-- 顶部栏：本地模型选择器与清空按钮 -->
    <div class="chat-header">
      <div class="model-selector">
        <label for="model-select">当前模型：</label>
        <select id="model-select" v-model="selectedModel" :disabled="loading || models.length === 0">
          <option v-if="models.length === 0" value="">未检测到本地模型</option>
          <option v-for="m in models" :key="m.name" :value="m.name">
            {{ m.name }}
          </option>
        </select>
        <button class="icon-btn" title="刷新模型列表" :disabled="loading" @click="fetchLocalModels">
          🔄
        </button>
      </div>
      <button class="clear-btn" :disabled="loading" @click="clearChat">清空对话</button>
    </div>

    <!-- 消息列表区域 -->
    <div ref="chatBoxRef" class="chat-box">
      <div
        v-for="(msg, index) in messages"
        :key="index"
        :class="['message-row', msg.role === 'user' ? 'user-row' : 'ai-row']"
      >
        <!-- 用户消息：纯文本展示 -->
        <div v-if="msg.role === 'user'" class="message-bubble user-bubble">
          {{ msg.content }}
        </div>

        <!-- AI 消息：Markdown 渲染 + 代码高亮 -->
        <div v-else class="message-bubble ai-bubble">
          <div class="markdown-body" v-html="renderMarkdown(msg.content)"></div>
          <span v-if="loading && index === messages.length - 1" class="cursor">▍</span>
        </div>
      </div>
    </div>

    <!-- 底部输入框与控制按钮 -->
    <div class="input-area">
      <input
        v-model="input"
        type="text"
        placeholder="输入消息，按 Enter 发送..."
        :disabled="loading || !selectedModel"
        @keydown.enter="handleSend"
      />
      <button
        v-if="!loading"
        class="send-btn"
        :disabled="!input.trim() || !selectedModel"
        @click="handleSend"
      >
        发送
      </button>
      <button v-else class="stop-btn" @click="handleStop">
        停止
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import MarkdownIt from 'markdown-it';
import hljs from 'highlight.js';
// 引入代码高亮主题样式（可根据喜好替换为 atom-one-dark / vs2015 等）
import 'highlight.js/styles/github-dark.css';

const OLLAMA_BASE_URL = 'http://localhost:11434';

// 初始化 MarkdownIt 与代码高亮
const md = new MarkdownIt({
  html: false,       // 禁用 HTML 标签防 XSS
  linkify: true,     // 自动识别链接
  breaks: true,      // 换行符转为 <br>
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return `<pre class="hljs"><code>${hljs.highlight(str, { language: lang, ignoreIllegals: true }).value}</code></pre>`;
      } catch (__) {}
    }
    return `<pre class="hljs"><code>${md.utils.escapeHtml(str)}</code></pre>`;
  }
});

// 渲染 Markdown 函数
const renderMarkdown = (content) => {
  return md.render(content || '');
};

// 状态定义
const models = ref([]);
const selectedModel = ref('');
const messages = ref([
  { role: 'assistant', content: '你好！请选择顶部本地 Ollama 模型开始对话。支持 **Markdown** 与 `代码高亮` 输出。' }
]);
const input = ref('');
const loading = ref(false);
const chatBoxRef = ref(null);
let abortController = null;

// 获取本地已下载的 Ollama 模型列表
const fetchLocalModels = async () => {
  try {
    const res = await fetch(`${OLLAMA_BASE_URL}/api/tags`);
    if (!res.ok) throw new Error('获取模型失败');
    const data = await res.json();
    models.value = data.models || [];
    // 默认选中第一个可用模型
    if (models.value.length > 0 && !models.value.some(m => m.name === selectedModel.value)) {
      selectedModel.value = models.value[0].name;
    }
  } catch (err) {
    console.error('无法连接 Ollama 获取模型:', err);
  }
};

onMounted(() => {
  fetchLocalModels();
});

const scrollToBottom = async () => {
  await nextTick();
  if (chatBoxRef.value) {
    chatBoxRef.value.scrollTop = chatBoxRef.value.scrollHeight;
  }
};

const clearChat = () => {
  messages.value = [];
};

const handleStop = () => {
  if (abortController) {
    abortController.abort();
    abortController = null;
    loading.value = false;
  }
};

const handleSend = async () => {
  const text = input.value.trim();
  if (!text || loading.value || !selectedModel.value) return;

  messages.value.push({ role: 'user', content: text });
  input.value = '';
  loading.value = true;
  await scrollToBottom();

  const contextMessages = messages.value.map(({ role, content }) => ({ role, content }));
  messages.value.push({ role: 'assistant', content: '' });
  const aiMsgIndex = messages.value.length - 1;

  abortController = new AbortController();

  try {
    const response = await fetch(`${OLLAMA_BASE_URL}/api/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      signal: abortController.signal,
      body: JSON.stringify({
        model: selectedModel.value,
        messages: contextMessages,
        stream: true
      })
    });

    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    const reader = response.body.getReader();
    const decoder = new TextDecoder('utf-8');
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop();

      for (const line of lines) {
        if (!line.trim()) continue;
        try {
          const json = JSON.parse(line);
          if (json.message?.content) {
            messages.value[aiMsgIndex].content += json.message.content;
            scrollToBottom();
          }
          if (json.done) break;
        } catch (e) {
          console.error('JSON 解析失败:', e);
        }
      }
    }
  } catch (error) {
    if (error.name === 'AbortError') {
      messages.value[aiMsgIndex].content += '\n\n*[已停止生成]*';
    } else {
      messages.value[aiMsgIndex].content = `**请求出错**：${error.message}（请确认已设置 \`OLLAMA_ORIGINS="*"\` 并启动 Ollama）`;
    }
  } finally {
    loading.value = false;
    abortController = null;
    scrollToBottom();
  }
};
</script>

<style scoped>
.chat-container {
  max-width: 780px;
  margin: 24px auto;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  height: 640px;
  background-color: #fff;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}

/* 顶部栏样式 */
.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background-color: #ffffff;
  border-bottom: 1px solid #e5e7eb;
}

.model-selector {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  color: #374151;
}

select {
  padding: 6px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  background-color: #f9fafb;
  outline: none;
}

.icon-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 15px;
  padding: 4px;
}

.clear-btn {
  background-color: #f3f4f6;
  color: #4b5563;
  border: 1px solid #d1d5db;
  padding: 5px 12px;
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
}

.clear-btn:hover:not(:disabled) {
  background-color: #e5e7eb;
}

/* 消息区样式 */
.chat-box {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
  background-color: #f9fafb;
}

.message-row {
  display: flex;
  margin-bottom: 18px;
}

.user-row {
  justify-content: flex-end;
}

.ai-row {
  justify-content: flex-start;
}

.message-bubble {
  max-width: 82%;
  padding: 12px 16px;
  border-radius: 10px;
  line-height: 1.6;
  font-size: 15px;
  word-break: break-word;
}

.user-bubble {
  background-color: #2563eb;
  color: #ffffff;
  white-space: pre-wrap;
  border-bottom-right-radius: 2px;
}

.ai-bubble {
  background-color: #ffffff;
  color: #1f2937;
  border: 1px solid #e5e7eb;
  border-bottom-left-radius: 2px;
}

/* Markdown 内部排版优化 */
.markdown-body :deep(p) {
  margin: 0 0 10px 0;
}
.markdown-body :deep(p:last-child) {
  margin-bottom: 0;
}
.markdown-body :deep(pre.hljs) {
  padding: 12px;
  border-radius: 8px;
  overflow-x: auto;
  margin: 10px 0;
  font-size: 13px;
  line-height: 1.5;
}
.markdown-body :deep(code) {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
}
.markdown-body :deep(:not(pre) > code) {
  background-color: #f3f4f6;
  color: #d946ef;
  padding: 2px 5px;
  border-radius: 4px;
  font-size: 0.9em;
}
.markdown-body :deep(ul), .markdown-body :deep(ol) {
  padding-left: 20px;
  margin: 8px 0;
}

.cursor {
  display: inline-block;
  margin-left: 2px;
  animation: blink 1s infinite;
}

@keyframes blink {
  0%, 50% { opacity: 1; }
  51%, 100% { opacity: 0; }
}

/* 输入框区域 */
.input-area {
  display: flex;
  padding: 12px 16px;
  background-color: #ffffff;
  border-top: 1px solid #e5e7eb;
  gap: 10px;
}

input {
  flex: 1;
  padding: 10px 14px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  outline: none;
  font-size: 14px;
}

input:focus {
  border-color: #2563eb;
}

.send-btn, .stop-btn {
  padding: 0 20px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
  color: #fff;
}

.send-btn {
  background-color: #2563eb;
}
.send-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.stop-btn {
  background-color: #dc2626;
}
</style>
