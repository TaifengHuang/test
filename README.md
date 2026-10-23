<template>
  <div class="app-container chat-container" :class="{ 'has-evidence-panel': activeSourceMessage }">
    <!-- 每次进入都显示的欢迎页 -->
    <div v-if="showWelcome" class="welcome-page" v-loading="startingChat" :class="{ 'is-starting': startingChat }">
      <el-button type="primary" size="large" class="start-chat-btn" @click="startChat">开始聊天</el-button>
    </div>

    <!-- 会话界面 -->
    <el-container v-else style="height: 100%">
      <!-- 侧边栏：会话历史 -->
      <el-aside v-show="!sidebarCollapsed && !activeSourceMessage" width="340px" class="session-sidebar">
        <!-- 侧边栏顶部操作区 -->
        <div class="sidebar-top-actions">
          <!-- 聊天模式下拉选择 -->
          <el-select v-model="chatMode" :disabled="loading" aria-label="聊天模式" class="mode-select" @change="changeChatMode" title="切换模式会新建会话">
            <el-option value="chat" label="普通聊天" />
            <el-option value="product" label="参数问答" />
          </el-select>

          <div class="top-icons">
            <!-- 收起侧边栏图标 -->
            <el-tooltip content="收起会话栏" placement="bottom">
              <el-button link class="icon-btn" @click="toggleSessionSidebar">
                <el-icon size="18"><Fold /></el-icon>
              </el-button>
            </el-tooltip>
          </div>
        </div>

        <!-- 新建对话按钮 -->
        <div class="sidebar-header">
          <el-button class="new-chat-btn" @click="newConversation">
            <el-icon class="btn-icon"><Plus /></el-icon> 新建对话
          </el-button>
        </div>

        <!-- 会话搜索框：直接显示在新建对话下方 -->
        <div class="sidebar-search">
          <el-input
            v-model="sessionSearch"
            placeholder="搜索历史对话..."
            clearable
            :prefix-icon="Search"
            aria-label="搜索历史对话"
          />
        </div>

        <div v-if="sessionError" role="status" class="session-load-error">
          <p>{{ sessionError }}</p>
          <el-button size="small" :loading="sessionLoading" @click="getSessions">重试读取</el-button>
        </div>

        <!-- 会话列表 -->
        <div class="session-list" v-loading="sessionLoading">
          <div v-if="filteredSessions.length > 0" class="session-group-label">今天</div>

          <div v-for="session in filteredSessions" :key="session.sessionId" :class="['session-item', currentSessionId === session.sessionId ? 'active' : '']">
            <button type="button" class="session-open" :aria-label="`打开会话：${session.sessionTitle || '新对话'}`" :aria-pressed="currentSessionId === session.sessionId" @click="loadSession(session.sessionId)">
              <span class="session-title">{{ session.sessionTitle || "新对话" }}</span>
            </button>
            <div class="session-actions">
              <el-button link icon="Edit" :aria-label="`重命名`" :disabled="!!renamingSessionId" @click.stop="handleRenameSession(session)" />
              <el-button class="delete-btn" type="danger" link icon="Delete" :aria-label="`删除`" @click.stop="handleDeleteSession(session.sessionId)" />
            </div>
          </div>
          <div v-if="filteredSessions.length === 0 && !sessionLoading && !sessionError" class="empty-session">
            没有符合搜索的会话
          </div>
        </div>
      </el-aside>

      <!-- 主区域：对话框 -->
      <el-main class="chat-main">
        <div class="chat-header">
          <div class="header-left">
            <!-- 当侧边栏收起时，提供一个展开按钮 -->
            <el-button v-if="sidebarCollapsed" link class="expand-btn" @click="toggleSessionSidebar">
              <el-icon size="18"><Expand /></el-icon>
            </el-button>
            <span class="header-title">{{ currentSessionTitle }}</span>
          </div>
          <div class="header-right-actions">
            <div v-if="chatMode === 'product'" class="product-header-actions">
              <el-tooltip content="全局参数配置" placement="bottom">
                <el-button icon="Setting" aria-label="全局参数配置" circle style="margin-right: 10px" @click="openConfigDialog"></el-button>
              </el-tooltip>
              <el-select v-model="currentModelId" placeholder="选择模型" aria-label="选择聊天模型" :loading="modelLoading" :disabled="modelLoading || loading" size="large" style="width: 210px">
                <el-option v-for="item in modelOptions" :key="item.modelId" :label="`${item.provider}/${item.modelCode}`" :value="item.modelId" />
              </el-select>
            </div>
          </div>
        </div>

        <div v-if="chatMode === 'product' && (modelError || (modelsLoaded && !modelOptions.length))" role="status" class="session-load-error">
          <p>{{ modelError || '当前没有可选择的模型，请联系管理员配置后刷新。' }}</p>
          <el-button :loading="modelLoading" @click="getModels">重新读取模型</el-button>
        </div>
        <div v-if="chatMode === 'product'" class="product-scope">
          <el-select v-model="productLibraryIds" multiple filterable :disabled="loading || productLibrariesLoading" placeholder="选择产品知识库" aria-label="选择产品知识库" style="min-width:260px" @change="selectedProductId = null">
            <el-option v-for="library in productLibraries" :key="library.libraryId" :value="library.libraryId" :label="library.name" :disabled="library.status !== 'ACTIVE'" />
          </el-select>
          <el-button :loading="productLibrariesLoading" :disabled="loading" @click="loadProductLibraries">刷新产品知识库</el-button>
          <span v-if="productLibrariesError" role="status">{{ productLibrariesError }}</span>
          <span>检索所选授权产品知识库并结合知识库返回结果统一回答</span>
          <el-button v-if="selectedProductId" :disabled="loading" @click="selectedProductId = null">清除产品 {{ selectedProductId }}</el-button>
        </div>
        <el-collapse v-if="chatMode === 'product'" class="product-query-conditions">
          <el-collapse-item title="适用条件（可选，对下一轮生效）" name="conditions">
            <el-select v-model="queryConditionMode" aria-label="本轮适用条件选择" :disabled="loading">
              <el-option value="any" label="未限定条件" /><el-option value="none" label="明确无附加条件" /><el-option value="specified" label="指定适用条件" />
              <el-option v-if="queryHistoryConditions" value="history" label="沿用历史复合条件" />
            </el-select>
            <p>问题涉及温度、配置等条件时，请在此明确填写；未填写时不会推断。</p>
            <p v-if="queryConditionMode === 'history'">{{ conditionText(queryHistoryConditions) }}。重新指定或清除后可修改。</p>
            <template v-if="queryConditionMode === 'specified'">
              <div v-for="(row, index) in queryConditionRows" :key="index" class="query-condition-row">
                <el-input v-model="row.key" :disabled="loading" maxlength="100" placeholder="例如：环境温度" :aria-label="`查询条件 ${index + 1} 名称`" />
                <el-select v-model="row.type" :disabled="loading" :aria-label="`查询条件 ${index + 1} 类型`" @change="row.value = row.type === 'boolean' ? null : ''">
                  <el-option value="text" label="文本 / 精确数值" /><el-option value="boolean" label="是 / 否" />
                </el-select>
                <el-select v-if="row.type === 'boolean'" v-model="row.value" :disabled="loading" :aria-label="`查询条件 ${index + 1} 值`">
                  <el-option :value="true" label="是" /><el-option :value="false" label="否" />
                </el-select>
                <el-input v-else v-model="row.value" :disabled="loading" maxlength="500" placeholder="例如：40 ℃" :aria-label="`查询条件 ${index + 1} 值`" />
                <el-button :disabled="loading" @click="queryConditionRows.splice(index, 1)">移除</el-button>
              </div>
              <el-button :disabled="loading" @click="queryConditionRows.push({ key: '', type: 'text', value: '' })">添加条件</el-button>
            </template>
          </el-collapse-item>
        </el-collapse>
        <div v-if="sessionDetailError" role="status" class="session-load-error">
          <p>{{ sessionDetailError }}</p><el-button :loading="loading" @click="loadSession(currentSessionId)">重试读取当前会话</el-button>
        </div>
        <div class="chat-history" ref="chatHistoryRef" @scroll="handleScroll">
          <div class="chat-content" ref="chatContentRef" :class="{ 'is-empty': messageList.length === 0 }">
            <div v-if="messageList.length === 0" class="welcome-screen">
              <div class="welcome-icon"><el-icon size="60"><Service /></el-icon></div>
              <h2>你好！我是你的 AI 助手</h2>
              <p>请在下方输入问题开始对话...</p>
            </div>

            <div v-for="(msg, index) in messageList" :key="index" :class="['message-row', msg.role === 'user' ? 'message-user' : 'message-ai']">
              <div class="message-avatar">
                <el-avatar :icon="msg.role === 'user' ? 'UserFilled' : 'Service'" :size="40" :class="msg.role === 'user' ? 'avatar-user' : 'avatar-ai'"></el-avatar>
              </div>
              <div class="message-content-wrapper">
                <div class="message-sender">
                  {{ msg.role === "user" ? "我" : "AI 助手" }}
                  <span class="message-time" v-if="msg.createdAt">{{ formatTime(msg.createdAt) }}</span>
                </div>
                <div class="message-bubble">
                  <div v-if="msg.role === 'user'">
                    <div v-if="msg.images && msg.images.length > 0" class="user-images">
                      <ChatImage v-for="(img, idx) in msg.images" :key="img + ':' + idx" :image="img" :session-id="currentSessionId || ''" :message-id="msg.id || ''" class="user-image-item" />
                    </div>
                    <div class="user-text">{{ msg.content }}</div>
                  </div>
                  <template v-else>
                    <ProductMessage v-if="msg.mode === 'product'" :message="msg" :source-active="activeSourceMessage === msg" @source-panel-change="setSourcePanel(msg, $event)" :loading="loading" :streaming="loading && index === messageList.length - 1" @choose="chooseProduct($event, msg)" @recover="recoverProduct(msg)" @stop="stopProductMessage(msg)" @retry="retryProduct(msg)" @retry-supplement="retryProduct(msg, true)" @access-lost="hideProductMessage(msg)" />
                    <AiMessage v-else :content="msg.content" :reasoning-content="msg.reasoningContent" :reasoning-status="msg.mode === 'product' ? msg.reasoningStatus : undefined" :loading="loading && index === messageList.length - 1" />
                  </template>
                  <div v-if="msg.role === 'assistant' && msg.evidenceImages?.length" class="ragflow-evidence">
                    <figure v-for="(evidence, evidenceIndex) in msg.evidenceImages" :key="evidence.id" class="ragflow-evidence-item">
                      <ChatImage :image="evidence.id" evidence :session-id="currentSessionId || ''" :message-id="msg.id || ''" class="ragflow-evidence-image" />
                      <figcaption>
                        <span>证据 {{ evidenceIndex + 1 }}</span>
                        <span v-if="evidence.content" class="ragflow-evidence-excerpt">{{ evidence.content }}</span>
                      </figcaption>
                    </figure>
                  </div>
                </div>
                <div v-if="chatMode === 'chat' && msg.id" class="message-footer">
                  <el-tooltip content="删除消息" placement="top">
                    <el-button link type="danger" :icon="Delete" aria-label="删除消息" size="small" :disabled="loading" @click="handleDeleteMessage(msg)"></el-button>
                  </el-tooltip>
                </div>
                <div class="message-footer">
                  <div class="footer-actions">
                    <el-tooltip content="复制" placement="top">
                      <el-button link type="info" :icon="DocumentCopy" aria-label="复制消息" size="small" @click="copyText(messageCopyText(msg))"></el-button>
                    </el-tooltip>
                    <div v-if="userConfig.metricsDefaultVisible == '0' && hasMetrics(msg)" class="message-metrics">
                      <span v-if="msg.metrics?.duration !== null && msg.metrics?.duration !== undefined">耗时 {{ msg.metrics.duration.toFixed(3) }} s</span>
                      <span v-if="msg.metrics?.inputTokens !== null && msg.metrics?.inputTokens !== undefined">输入 {{ msg.metrics.inputTokens }} tokens</span>
                      <span v-if="msg.metrics?.outputTokens !== null && msg.metrics?.outputTokens !== undefined">输出 {{ msg.metrics.outputTokens }} tokens</span>
                      <span v-if="msg.metrics?.totalTokens !== null && msg.metrics?.totalTokens !== undefined">总 {{ msg.metrics.totalTokens }} tokens</span>
                      <span v-if="msg.metrics?.reasoningTokens !== null && msg.metrics?.reasoningTokens !== undefined">推理 {{ msg.metrics.reasoningTokens }} tokens</span>
                    </div>
                  </div>
                  <div v-if="msg.role === 'assistant'" class="model-info">
                    <el-tag size="small" type="info" effect="plain" v-if="currentSessionAgentData?.model">{{ currentSessionAgentData.model.provider }} / {{ currentSessionAgentData.model.id }}</el-tag>
                    <el-tag size="small" type="info" effect="plain" v-else-if="currentModelInfo">{{ currentModelInfo.provider }} / {{ currentModelInfo.modelCode }}</el-tag>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 输入区域 -->
        <div class="chat-input-area">
          <div class="input-wrapper">
            <el-input
              v-model="inputMessage"
              type="textarea"
              :rows="2"
              resize="none"
              placeholder="请输入您的问题... (Enter 发送，Shift + Enter 换行)"
              @keydown.enter.exact.prevent="handleSend"
              :disabled="loading"
            />

            <!-- 已选知识库胶囊标签展示 -->
            <div v-if="chatMode === 'chat' && selectedKnowledgeBases.length > 0" class="selected-knowledge-bases">
              <div class="kb-pill" v-for="kb in selectedKnowledgeBases" :key="kb.libraryId">
                <span class="kb-name">{{ kb.name }}</span>
                <el-icon class="kb-close" @click="removeKnowledgeBase(kb.libraryId)"><Close /></el-icon>
              </div>
            </div>

            <!-- 产品模式图片展示 -->
            <div class="selected-images" v-if="chatMode === 'product' && userConfig.visionEnabled == '0' && inputImages.length">
              <div v-for="(img, idx) in inputImages" :key="img" class="selected-image-wrapper">
                <ChatImage :image="img" class="selected-image-item" />
                <el-button link :disabled="loading" :aria-label="`移除本轮第 ${idx + 1} 张图片`" @click="inputImages.splice(idx, 1)">移除</el-button>
              </div>
            </div>

            <!-- 底部操作栏 -->
            <div class="input-actions">
              <div class="left-actions">
                <el-button
                  v-if="chatMode === 'chat'"
                  class="add-kb-btn"
                  size="small"
                  :icon="Plus"
                  :disabled="loading || ragflowKnowledgeBasesLoading || ragflowKnowledgeBasesSaving"
                  @click="openKnowledgeBaseDialog"
                >
                  添加知识库
                </el-button>

                <el-tooltip v-if="chatMode === 'product' && currentModelInfo && currentModelInfo.supportImages === 'Y' && userConfig.visionEnabled == '0'" content="上传图片" placement="top">
                  <el-button circle plain size="small" :icon="Picture" @click="triggerImageUpload" />
                </el-tooltip>
                <el-button v-if="chatMode === 'product' && currentModelInfo && currentModelInfo.supportReasoning === 'Y'" class="toggle-chip" size="small" :type="chatConfig.isReasoning ? 'primary' : ''" :plain="!chatConfig.isReasoning" @click="chatConfig.isReasoning = !chatConfig.isReasoning">
                  <template #icon><svg-icon icon-class="deepthink" /></template>深度思考
                </el-button>
              </div>
              <el-button class="send-btn" :type="loading ? 'danger' : 'primary'" :icon="loading ? 'VideoPause' : 'Promotion'" circle size="large" @click="handleMainAction" :disabled="!loading && !inputMessage.trim() && !inputImages.length" />
            </div>
          </div>
        </div>
      </el-main>
    </el-container>

    <!-- 搜索会话弹窗 -->
    <el-dialog v-model="searchDialogVisible" title="搜索历史对话" width="400px" append-to-body class="search-dialog" @opened="focusSearchInput">
      <el-input
        ref="searchInputRef"
        v-model="sessionSearch"
        placeholder="请输入会话名称..."
        clearable
        :prefix-icon="Search"
        size="large"
      />
      <div class="search-hint" v-if="sessionSearch.trim()">
        找到 {{ filteredSessions.length }} 个相关会话
      </div>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="searchDialogVisible = false">关闭</el-button>
          <el-button type="primary" @click="searchDialogVisible = false">确定</el-button>
        </span>
      </template>
    </el-dialog>

    <!-- 知识库选择对话框 -->
    <el-dialog v-model="knowledgeBaseDialogVisible" title="添加知识库" width="480px" append-to-body class="kb-dialog">
      <div v-loading="ragflowKnowledgeBasesLoading" class="kb-dialog-content">
        <el-alert v-if="ragflowKnowledgeBasesError" :title="ragflowKnowledgeBasesError" type="error" show-icon :closable="false" style="margin-bottom: 16px;" />
        <div v-if="ragflowKnowledgeBases.length" class="kb-checkbox-list">
          <el-checkbox-group v-model="tempSelectedKnowledgeBaseIds">
            <el-checkbox v-for="kb in ragflowKnowledgeBases" :key="kb.libraryId" :label="kb.libraryId" :disabled="ragflowKnowledgeBasesSaving" class="kb-checkbox-item">
              <div class="kb-item-info">
                <span class="kb-item-name">{{ kb.name }}</span>
                <span class="kb-item-desc" v-if="kb.description">{{ kb.description }}</span>
              </div>
            </el-checkbox>
          </el-checkbox-group>
        </div>
        <el-empty v-else-if="!ragflowKnowledgeBasesLoading" description="暂无可用的知识库" :image-size="80" />
      </div>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="knowledgeBaseDialogVisible = false">取消</el-button>
          <el-button type="primary" :loading="ragflowKnowledgeBasesSaving" :disabled="ragflowKnowledgeBasesLoading" @click="confirmKnowledgeBaseSelection">确定</el-button>
        </span>
      </template>
    </el-dialog>

    <!-- 全局配置弹窗 -->
    <el-dialog v-model="showConfigDialog" title="用户全局配置" width="700px" append-to-body class="chat-config-dialog">
      <p v-if="configError" role="alert">{{ configError }}</p>
      <el-button v-if="!configLoaded" :loading="configLoading" @click="loadUserConfig">重新读取配置</el-button>
      <el-form :model="editingUserConfig" label-width="150px" :disabled="!configLoaded || configLoading || configSaving">
        <el-row :gutter="20">
          <el-col :span="12"><el-form-item label="默认温度"><el-input-number v-model="editingUserConfig.temperature" :min="0" :max="2" :step="0.1" :precision="1" placeholder="默认温度" style="width: 100%" /></el-form-item></el-col>
          <el-col :span="12"><el-form-item label="附带历史消息"><el-switch active-value="0" inactive-value="1" v-model="editingUserConfig.addHistoryToContext" /></el-form-item></el-col>
          <el-col :span="12"><el-form-item label="历史消息轮数" v-if="editingUserConfig.addHistoryToContext == '0'"><el-input-number v-model="editingUserConfig.numHistoryRuns" :min="1" :max="20" placeholder="历史消息轮数" style="width: 100%" /></el-form-item></el-col>
          <el-col :span="12"><el-form-item label="默认显示指标"><el-switch active-value="0" inactive-value="1" v-model="editingUserConfig.metricsDefaultVisible" /></el-form-item></el-col>
          <el-col :span="12"><el-form-item label="开启视觉功能"><el-switch active-value="0" inactive-value="1" v-model="editingUserConfig.visionEnabled" /></el-form-item></el-col>
          <el-col :span="12"><el-form-item label="图片最大大小" v-if="editingUserConfig.visionEnabled === '0'"><el-input-number v-model="editingUserConfig.imageMaxSizeMb" :min="1" :max="50" placeholder="图片大小" style="width: 100%"><template #suffix><span>MB</span></template></el-input-number></el-form-item></el-col>
          <el-col :span="24"><el-form-item label="系统提示词"><el-input v-model="editingUserConfig.systemPrompt" type="textarea" :rows="4" placeholder="设置全局系统提示词" /></el-form-item></el-col>
        </el-row>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="showConfigDialog = false">取消</el-button>
          <el-button type="primary" :loading="configSaving" :disabled="!configLoaded || configLoading" @click="handleSaveConfig">保存</el-button>
        </span>
      </template>
    </el-dialog>
    <input v-if="userConfig.visionEnabled === '0'" ref="imageInputRef" type="file" accept="image/*" multiple class="chat-image-input" @change="handleImageInputChange" />
  </div>
</template>

<script setup name="AiChat">
import { ElMessageBox } from 'element-plus';
import { Plus, Close, Expand, Fold, Search } from '@element-plus/icons-vue';
import useUserStore from '@/store/modules/user';
import { listModelAll } from "@/api/ai/model";
import {
  listChatSession, delChatSession, renameChatSession, getChatSession,
  getUserChatConfig, saveUserChatConfig, cancelChatRun, getProductGeneration,
  startRagflowChat, getRagflowKnowledgeBases, updateRagflowKnowledgeBases,
  listRagflowChatSessions, createRagflowChatSession, getRagflowChatSession,
  renameRagflowChatSession, deleteRagflowChatSessions, deleteRagflowChatMessage,
} from "@/api/ai/chat";
import { getToken } from "@/utils/auth";
import AiMessage from "./components/AiMessage.vue";
import ProductMessage from './components/ProductMessage.vue';
import ChatImage from './components/ChatImage.vue';
import { conditionText } from '@/views/bid/productParameters/presentation';
import { messageCopyText } from './messageCopy';
import { applyProductEvent, restoreProductEvents } from './productEvents';
import { listLibraries } from '@/api/bid/product';
import { Picture, DocumentCopy, Delete } from "@element-plus/icons-vue";
import { v4 as uuidv4 } from "uuid";
import { useResizeObserver } from "@vueuse/core";
import { getUseMonaco } from 'markstream-vue'

getUseMonaco()

const { proxy } = getCurrentInstance();

// ===== 欢迎页状态 =====
const showWelcome = ref(true);
const startingChat = ref(false);
const ragflowReady = ref(false);
const ragflowKnowledgeBases = ref([]);
const ragflowLibraryIds = ref([]);
const ragflowKnowledgeBasesLoading = ref(false);
const ragflowKnowledgeBasesSaving = ref(false);
const ragflowKnowledgeBasesError = ref('');
let ragflowKnowledgeBasesRequest = 0;

// ===== 搜索弹窗 =====
const searchDialogVisible = ref(false);
const searchInputRef = ref(null);
function openSearchDialog() {
  searchDialogVisible.value = true;
}
function focusSearchInput() {
  if (searchInputRef.value) {
    searchInputRef.value.focus();
  }
}

// ===== 知识库选择弹窗相关 =====
const knowledgeBaseDialogVisible = ref(false);
const tempSelectedKnowledgeBaseIds = ref([]);

const selectedKnowledgeBases = computed(() => {
  return ragflowKnowledgeBases.value.filter(kb => ragflowLibraryIds.value.includes(kb.libraryId));
});

function openKnowledgeBaseDialog() {
  if (ragflowKnowledgeBasesLoading.value) return;
  tempSelectedKnowledgeBaseIds.value = [...ragflowLibraryIds.value];
  knowledgeBaseDialogVisible.value = true;
}

async function confirmKnowledgeBaseSelection() {
  ragflowLibraryIds.value = [...tempSelectedKnowledgeBaseIds.value];
  knowledgeBaseDialogVisible.value = false;
  await saveRagflowKnowledgeBases();
}

function removeKnowledgeBase(libraryId) {
  ragflowLibraryIds.value = ragflowLibraryIds.value.filter(id => id !== libraryId);
  saveRagflowKnowledgeBases();
}

async function startChat() {
  if (startingChat.value) return;
  const owner = accountGeneration;
  startingChat.value = true;
  try {
    await startRagflowChat();
    if (owner !== accountGeneration) return;
    ragflowReady.value = true;
    await loadRagflowKnowledgeBases();
    if (owner !== accountGeneration) return;
    await getSessions();
    if (owner !== accountGeneration) return;
    const sessionId = await ensureRagflowSession();
    if (owner !== accountGeneration) return;
    showWelcome.value = false;
    await loadSession(sessionId, true);
  } catch (error) {
    if (owner === accountGeneration) {
      showWelcome.value = true;
      proxy.$modal.msgError(error?.response?.data?.detail || error?.message || '开始聊天失败');
    }
  } finally {
    if (owner === accountGeneration) startingChat.value = false;
  }
}
// =====================

const account = useUserStore();
let accountGeneration = 0;
const modelOptions = ref([]), modelLoading = ref(false), modelsLoaded = ref(false), modelError = ref('');
let modelRequest = 0;
const currentModelId = ref(undefined);
const chatMode = ref('chat'), productLibraryIds = ref([]), productLibraries = ref([]), selectedProductId = ref(null);
const productLibrariesLoading = ref(false), productLibrariesError = ref('');
let productLibraryRequest;
let chatGeneration = 0;
function invalidateStream() {
  activeSourceMessage.value = null;
  productLibraryRequest?.abort(); productLibrariesLoading.value = false;
  const message = messageList.value[messageList.value.length - 1];
  if (message?.mode === 'product' && message.productStatus === 'running') message.productStatus = 'interrupted';
  chatGeneration++; abortController.value?.abort(); abortController.value = null; loading.value = false; currentRunId.value = null; }
async function changeChatMode() {
  clearChat();
  if (chatMode.value === 'product') {
    await getSessions();
    loadProductLibraries();
    return;
  }
  inputImages.value = [];
  try {
    await loadRagflowKnowledgeBases();
    await getSessions();
    await ensureRagflowSession();
    await loadSession(currentSessionId.value, true);
  } catch (error) {
    proxy.$modal.msgError(error?.response?.data?.detail || '切换聊天模式失败，请稍后重试');
  }
}

function applyRagflowKnowledgeBases(data) {
  ragflowKnowledgeBases.value = data?.options || [];
  ragflowLibraryIds.value = data?.selectedLibraryIds || [];
}

async function loadRagflowKnowledgeBases() {
  if (chatMode.value !== 'chat' || !ragflowReady.value) return;
  const owner = accountGeneration, request = ++ragflowKnowledgeBasesRequest;
  const current = () => owner === accountGeneration && request === ragflowKnowledgeBasesRequest;
  ragflowKnowledgeBasesLoading.value = true;
  ragflowKnowledgeBasesError.value = '';
  try {
    const { data } = await getRagflowKnowledgeBases();
    if (current()) applyRagflowKnowledgeBases(data);
  } catch (error) {
    if (current()) {
      ragflowKnowledgeBases.value = [];
      ragflowLibraryIds.value = [];
      ragflowKnowledgeBasesError.value = error?.response?.data?.detail || '知识库读取失败，请刷新后重试';
    }
  } finally {
    if (current()) ragflowKnowledgeBasesLoading.value = false;
  }
}

async function saveRagflowKnowledgeBases() {
  if (ragflowKnowledgeBasesSaving.value || ragflowKnowledgeBasesLoading.value) return;
  const owner = accountGeneration, request = ++ragflowKnowledgeBasesRequest;
  ragflowKnowledgeBasesSaving.value = true;
  ragflowKnowledgeBasesError.value = '';
  try {
    const selected = [...ragflowLibraryIds.value];
    const { data } = await updateRagflowKnowledgeBases(selected);
    if (owner === accountGeneration && request === ragflowKnowledgeBasesRequest) {
      applyRagflowKnowledgeBases(data);
      proxy.$modal.msgSuccess('知识库范围已更新');
    }
  } catch (error) {
    if (owner === accountGeneration && request === ragflowKnowledgeBasesRequest) {
      ragflowKnowledgeBasesError.value = error?.response?.data?.detail || '知识库更新失败，请稍后重试';
      await loadRagflowKnowledgeBases();
    }
  } finally {
    if (owner === accountGeneration) ragflowKnowledgeBasesSaving.value = false;
  }
}

async function loadProductLibraries() {
  if (chatMode.value !== 'product') return;
  productLibraryRequest?.abort();
  const request = productLibraryRequest = new AbortController(), epoch = chatGeneration;
  const selected = [...productLibraryIds.value];
  productLibraries.value = []; productLibraryIds.value = [];
  productLibrariesLoading.value = true; productLibrariesError.value = '';
  try {
    const rows = []; let page = 1;
    while (true) {
      const { data } = await listLibraries({ page, pageSize: 100 }, request.signal);
      if (request.signal.aborted || epoch !== chatGeneration) return;
      rows.push(...data.rows);
      if (page * 100 >= data.total) break;
      page++;
    }
    productLibraries.value = rows;
    const allowed = new Set(rows.filter(row => row.status === 'ACTIVE').map(row => row.libraryId));
    productLibraryIds.value = selected.filter(id => allowed.has(id));
    if (productLibraryIds.value.length !== selected.length) selectedProductId.value = null;
    if (!allowed.size) productLibrariesError.value = '暂无可用于查询的资料库，请确认权限和资料库状态。';
  } catch {
    if (!request.signal.aborted && epoch === chatGeneration) {
      selectedProductId.value = null;
      productLibrariesError.value = '资料库读取失败，请检查网络和权限后刷新。';
    }
  } finally { if (!request.signal.aborted && epoch === chatGeneration) productLibrariesLoading.value = false; }
}
async function chooseProduct(id, message) {
  if (loading.value || productLibrariesLoading.value || sessionDetailError.value) return;
  if (message.sessionId !== currentSessionId.value || !message.requestMessage || !message.libraryIds?.length
      || message.parameterResult?.productStatus !== 'NEEDS_CONFIRMATION'
      || !message.parameterResult.choices?.some(choice => choice.productId === id)) return;
  if (inputMessage.value.trim() || inputImages.value.length) {
    proxy.$modal.msgError('请先处理当前输入草稿，再确认历史问题的产品'); return;
  }
  if (!currentModelId.value) { proxy.$modal.msgError('请先选择模型'); return; }
  chatMode.value = 'product'; productLibraryIds.value = [...message.libraryIds]; selectedProductId.value = id;
  inputMessage.value = message.requestMessage; inputImages.value = [...(message.requestImages || [])]; restoreQueryConditions(message.requestConditions); await handleSend();
}
async function retryProduct(message, supplementOnly = false) {
  if (loading.value || productLibrariesLoading.value || sessionDetailError.value) return;
  if (!(supplementOnly ? message.productStatus === 'completed' && message.supplementFailed : ['failed', 'stopped'].includes(message.productStatus)) || !message.generationId
      || message.sessionId !== currentSessionId.value || !message.requestMessage || !message.libraryIds?.length) return;
  if (inputMessage.value.trim() || inputImages.value.length) {
    proxy.$modal.msgError('请先处理当前输入草稿，再重新查询历史问题'); return;
  }
  if (!currentModelId.value) { proxy.$modal.msgError('请先选择模型'); return; }
  chatMode.value = 'product'; productLibraryIds.value = [...message.libraryIds];
  selectedProductId.value = message.parameterResult?.productStatus === 'CONFIRMED' ? message.parameterResult.productId : null;
  inputMessage.value = message.requestMessage; inputImages.value = [...(message.requestImages || [])]; restoreQueryConditions(message.requestConditions);
  await handleSend(undefined, supplementOnly ? message.generationId : null);
}
function hideProductMessage(message) {
  Object.assign(message, { content: '', reasoningContent: '', reasoningStatus: 'completed', references: [], parameterResult: null, currentParameterStatuses: {}, productStatus: 'failed', productErrors: ['此轮资料当前不可访问，请确认权限。'] });
}
async function stopProductMessage(message) {
  if (loading.value || !message.generationId || message.sessionId !== currentSessionId.value) return;
  const epoch = chatGeneration;
  loading.value = true;
  try {
    await cancelChatRun(message.runId || message.generationId, message.sessionId, message.generationId);
    if (epoch === chatGeneration) await recoverProduct(message);
  } catch {
    if (epoch === chatGeneration) message.productErrors = ['结束状态尚未确认，请先恢复本轮结果再重试。'];
  } finally { if (epoch === chatGeneration) loading.value = false; }
}
async function recoverProduct(message) {
  const epoch = chatGeneration;
  try {
    const { data } = await getProductGeneration(message.sessionId, message.generationId);
    if (epoch === chatGeneration) {
      if (data.sessionId !== message.sessionId || data.generationId !== message.generationId) throw new Error('历史身份不匹配');
      Object.assign(message, restoreProductEvents(data));
    }
  } catch (error) {
    if (epoch !== chatGeneration) return;
    if ([401, 403, 404].includes(error?.response?.status)) { hideProductMessage(message); }
    message.productErrors = ['无法读取本轮结果，请检查资料权限或稍后重试。'];
  }
}
onBeforeUnmount(invalidateStream);
onDeactivated(invalidateStream);
const messageList = ref([]);
const inputMessage = ref("");
const inputImages = ref([]);
const queryConditionMode = ref('any'), queryConditionRows = ref([]), queryHistoryConditions = ref(null);
function restoreQueryConditions(value) {
  queryHistoryConditions.value = value ? JSON.parse(JSON.stringify(value)) : null;
  const fields = Object.entries(value || {}).filter(([key]) => key !== 'unknown');
  queryConditionRows.value = fields.filter(([, item]) => typeof item === 'string' || typeof item === 'boolean').map(([key, item]) => ({ key, value: item, type: typeof item === 'boolean' ? 'boolean' : 'text' }));
  queryConditionMode.value = !value ? 'any' : value.unknown || queryConditionRows.value.length !== fields.length ? 'history' : fields.length ? 'specified' : 'none';
}
function queryConditionsPayload() {
  if (queryConditionMode.value === 'any') return undefined;
  if (queryConditionMode.value === 'none') return { unknown: false };
  if (queryConditionMode.value === 'history') {
    const validate = value => {
      if (typeof value === 'number' && !Number.isSafeInteger(value)) throw new Error('历史条件中的数值无法安全重用，请重新填写精确文本后查询。');
      if (value && typeof value === 'object') Object.values(value).forEach(validate);
    };
    validate(queryHistoryConditions.value);
    return JSON.parse(JSON.stringify(queryHistoryConditions.value));
  }
  if (!queryConditionRows.value.length) throw new Error('请填写具体条件，或选择未限定条件。');
  const result = { unknown: false };
  for (const row of queryConditionRows.value) {
    const key = row.key.trim(), value = row.type === 'boolean' ? row.value : String(row.value).trim();
    if (!key || ['unknown', '__proto__', 'prototype', 'constructor'].includes(key) || Object.hasOwn(result, key)) throw new Error('条件名称不能为空、重复或使用保留名称。');
    if (row.type === 'boolean' ? typeof value !== 'boolean' : !value) throw new Error('请填写每条条件的值。');
    result[key] = value;
  }
  return result;
}

const loading = ref(false);
const chatHistoryRef = ref(null);
const chatContentRef = ref(null);
const currentSessionId = ref(null), sessionDetailError = ref('');
const showConfigDialog = ref(false);
const imageInputRef = ref(null);
const sessionList = ref([]);
const sessionSearch = ref(''), renamingSessionId = ref(null), sidebarCollapsed = ref(false);
const activeSourceMessage = shallowRef(null);
const filteredSessions = computed(() => {
  const query = sessionSearch.value.trim().normalize('NFKC').toLocaleLowerCase();
  return sessionList.value.filter(session => !query || (session.sessionTitle || '新对话').normalize('NFKC').toLocaleLowerCase().includes(query));
});
const currentSessionTitle = computed(() => sessionList.value.find(session => session.sessionId === currentSessionId.value)?.sessionTitle || (chatMode.value === 'product' ? '产品参数智能问答' : 'AI 智能助手'));
function setSourcePanel(message, open) {
  if (open) activeSourceMessage.value = message;
  else if (activeSourceMessage.value === message) activeSourceMessage.value = null;
}
function toggleSessionSidebar() {
  if (activeSourceMessage.value) { activeSourceMessage.value = null; sidebarCollapsed.value = false; }
  else sidebarCollapsed.value = !sidebarCollapsed.value;
}

const sessionLoading = ref(false), sessionError = ref('');
let sessionRequest = 0;
const abortController = ref(null);
const currentRunId = ref(null);
const isAutoScroll = ref(true);
const currentSessionAgentData = ref(null);
const isProgrammaticScroll = ref(false);
let scrollTimeout = null;

function generateSessionId() {
  return uuidv4();
}

const chatConfig = reactive({
  temperature: undefined,
  isReasoning: true,
});

const userConfig = reactive({
  chatConfigId: undefined,
  userId: undefined,
  temperature: undefined,
  addHistoryToContext: "0",
  numHistoryRuns: 3,
  systemPrompt: "",
  metricsDefaultVisible: "1",
  visionEnabled: "0",
  imageMaxSizeMb: 5,
  createTime: undefined,
  updateTime: undefined,
});

const editingUserConfig = reactive({
  chatConfigId: undefined,
  userId: undefined,
  temperature: undefined,
  addHistoryToContext: "0",
  numHistoryRuns: 3,
  systemPrompt: "",
  metricsDefaultVisible: "1",
  visionEnabled: "0",
  imageMaxSizeMb: 5,
  createTime: undefined,
  updateTime: undefined,
});

const defaultUserConfig = { ...userConfig };

const currentModelInfo = computed(() => {
  if (!currentModelId.value) return null;
  return modelOptions.value.find((m) => m.modelId === currentModelId.value);
});

const configLoading = ref(false), configLoaded = ref(false), configSaving = ref(false), configError = ref('');
let configRequest = 0;
async function loadUserConfig() {
  if (configSaving.value) return;
  const owner = accountGeneration, request = ++configRequest;
  configLoading.value = true; configLoaded.value = false; configError.value = '';
  try {
    const { data } = await getUserChatConfig();
    if (owner !== accountGeneration || request !== configRequest) return;
    Object.assign(userConfig, defaultUserConfig, data || {});
    Object.assign(editingUserConfig, userConfig);
    configLoaded.value = true;
  } catch {
    if (owner === accountGeneration && request === configRequest) configError.value = '配置读取失败，请重新读取后编辑，避免覆盖已有配置。';
  } finally {
    if (owner === accountGeneration && request === configRequest) configLoading.value = false;
  }
}

function openConfigDialog() {
  Object.assign(editingUserConfig, userConfig);
  showConfigDialog.value = true;
  if (!configLoaded.value && !configLoading.value) loadUserConfig();
}

async function handleSaveConfig() {
  if (!configLoaded.value || configLoading.value || configSaving.value) return;
  const owner = accountGeneration;
  const payload = { ...editingUserConfig };
  configSaving.value = true; configError.value = '';
  try {
    await saveUserChatConfig(payload);
    if (owner !== accountGeneration) return;
    Object.assign(userConfig, payload);
    proxy.$modal.msgSuccess("配置保存成功");
    showConfigDialog.value = false;
  } catch {
    if (owner === accountGeneration) configError.value = '配置保存未确认，已保留编辑内容，请核对后重试。';
  } finally {
    if (owner === accountGeneration) configSaving.value = false;
  }
}

function hasMetrics(msg) {
  const m = msg?.metrics;
  if (!m) return false;
  return (
    (m.inputTokens !== null && m.inputTokens !== undefined) ||
    (m.outputTokens !== null && m.outputTokens !== undefined) ||
    (m.totalTokens !== null && m.totalTokens !== undefined) ||
    (m.reasoningTokens !== null && m.reasoningTokens !== undefined) ||
    (m.duration !== null && m.duration !== undefined)
  );
}

function formatTime(timeStr) {
  if (!timeStr) return "";
  try {
    const date = new Date(timeStr);
    return date.toLocaleString();
  } catch (e) {
    return timeStr;
  }
}

async function getModels() {
  const owner = accountGeneration, request = ++modelRequest, selected = currentModelId.value;
  const current = () => owner === accountGeneration && request === modelRequest;
  modelLoading.value = true; modelsLoaded.value = false; modelError.value = '';
  modelOptions.value = []; currentModelId.value = undefined;
  try {
    const res = await listModelAll();
    if (!current()) return;
    modelOptions.value = res.data || []; modelsLoaded.value = true;
    const model = modelOptions.value.find(item => item.modelId === selected) || modelOptions.value[0];
    currentModelId.value = model?.modelId;
    chatConfig.temperature = model?.temperature;
  } catch {
    if (current()) modelError.value = '模型列表读取失败，请检查网络和登录状态后重试。';
  } finally {
    if (current()) modelLoading.value = false;
  }
}

watch(currentModelId, (newVal) => {
  const model = modelOptions.value.find((m) => m.modelId === newVal);
  if (model) {
    chatConfig.temperature = model.temperature;
  }
});

function defaultRagflowSessionName() {
  const now = new Date();
  const pad = value => String(value).padStart(2, '0');
  return `新对话 ${now.getFullYear()}-${pad(now.getMonth() + 1)}-${pad(now.getDate())} ${pad(now.getHours())}:${pad(now.getMinutes())}:${pad(now.getSeconds())}`;
}

async function ensureRagflowSession() {
  if (currentSessionId.value) return currentSessionId.value;
  const existing = sessionList.value[0];
  if (existing) {
    currentSessionId.value = existing.sessionId;
    return existing.sessionId;
  }
  const { data } = await createRagflowChatSession(defaultRagflowSessionName());
  sessionList.value = [data, ...sessionList.value];
  currentSessionId.value = data.sessionId;
  return data.sessionId;
}

async function newConversation() {
  if (chatMode.value === 'product') {
    clearChat();
    return;
  }
  const owner = accountGeneration;
  loading.value = true; sessionError.value = '';
  try {
    const { data } = await createRagflowChatSession(defaultRagflowSessionName());
    if (owner !== accountGeneration) return;
    sessionList.value = [data, ...sessionList.value];
    currentSessionId.value = data.sessionId;
    messageList.value = []; inputMessage.value = ''; inputImages.value = [];
    sessionDetailError.value = ''; currentSessionAgentData.value = null;
    activeSourceMessage.value = null;
  } catch (error) {
    if (owner === accountGeneration) {
      sessionError.value = error?.response?.data?.detail || '新建对话失败，请稍后重试';
    }
  } finally {
    if (owner === accountGeneration) loading.value = false;
  }
}

async function getSessions() {
  const owner = accountGeneration, request = ++sessionRequest;
  const current = () => owner === accountGeneration && request === sessionRequest;
  if (chatMode.value === 'chat' && !ragflowReady.value) return;
  sessionLoading.value = true; sessionError.value = ''; sessionList.value = [];
  try {
    const res = chatMode.value === 'chat'
      ? await listRagflowChatSessions()
      : await listChatSession();
    if (!current()) return;
    sessionList.value = [...(res.data || [])].sort((a, b) => new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime());
  } catch {
    if (current()) sessionError.value = '会话列表读取失败，请检查网络和登录状态后重试。';
  } finally {
    if (current()) sessionLoading.value = false;
  }
}

async function loadSession(sessionId, force = false) {
  if (!force && currentSessionId.value === sessionId && !sessionDetailError.value) return;
  if (chatMode.value === 'chat' && !ragflowReady.value) return;
  invalidateStream(); const epoch = chatGeneration;
  currentSessionId.value = sessionId; messageList.value = []; loading.value = true; selectedProductId.value = null;
  currentSessionAgentData.value = null; sessionDetailError.value = ''; productLibraryIds.value = [];
  try {
    const request = chatMode.value === 'chat'
      ? getRagflowChatSession(sessionId)
      : getChatSession(sessionId);
    const { data } = await request;
    if (epoch !== chatGeneration) return;
    messageList.value = data.messages || [];
    if (chatMode.value === 'chat') {
      currentSessionAgentData.value = null;
      isAutoScroll.value = true; scrollToBottom();
      return;
    }
    currentSessionAgentData.value = data.agentData;
    chatMode.value = data.productGenerationIds?.length ? 'product' : 'chat';
    for (const generationId of data.productGenerationIds || []) {
      try {
        const { data: record } = await getProductGeneration(sessionId, generationId);
        if (epoch !== chatGeneration) return;
        const message = restoreProductEvents(record);
        messageList.value.push({ role: 'user', content: message.requestMessage || '产品参数查询', images: message.requestImages || [] }, message);
        productLibraryIds.value = [...record.libraryIds];
        selectedProductId.value = message.parameterResult?.productStatus === 'CONFIRMED' ? message.parameterResult.productId : null;
      } catch {
        if (epoch !== chatGeneration) return;
        messageList.value.push({ role: 'assistant', mode: 'product', sessionId, generationId, content: '', productStatus: 'interrupted', productErrors: ['此轮结果当前不可读取，请确认资料权限或稍后重试。'] });
      }
    }
    if (epoch !== chatGeneration) return;
    loadProductLibraries(); isAutoScroll.value = true; scrollToBottom();
  } catch { if (epoch === chatGeneration) sessionDetailError.value = '会话读取失败，请检查网络和权限后重试，或新建对话。'; }
  finally { if (epoch === chatGeneration) loading.value = false; }
}

async function handleRenameSession(session) {
  const owner = accountGeneration;
  try {
    const { value } = await ElMessageBox.prompt('请输入会话名称', '重命名会话', {
      inputValue: session.sessionTitle || '', inputPlaceholder: '1–100 个字符',
      inputValidator: value => !!value?.trim() && value.trim().length <= 100 && !/[\u0000-\u001f\u007f]/.test(value) || '请输入 1–100 个字符且不含控制字符的名称',
      confirmButtonText: '保存', cancelButtonText: '取消',
    });
    if (owner !== accountGeneration) return;
    renamingSessionId.value = session.sessionId;
    if (chatMode.value === 'chat') {
      await renameRagflowChatSession(session.sessionId, value.trim());
    } else {
      await renameChatSession(session.sessionId, value.trim());
    }
    if (owner !== accountGeneration) return;
    const current = sessionList.value.find(item => item.sessionId === session.sessionId);
    if (current) current.sessionTitle = value.trim();
    proxy.$modal.msgSuccess('会话已重命名');
  } catch (error) {
    if (owner === accountGeneration && !['cancel', 'close'].includes(error)) proxy.$modal.msgError('会话名称保存未确认，请刷新列表核对后重试。');
  } finally { if (owner === accountGeneration) renamingSessionId.value = null; }
}

function handleDeleteSession(sessionId) {
  const owner = accountGeneration;
  proxy.$modal
    .confirm("是否确认删除该会话？")
    .then(function () {
      if (owner !== accountGeneration) throw new Error("账号已变化");
      return chatMode.value === 'chat'
        ? deleteRagflowChatSessions([sessionId])
        : delChatSession(sessionId);
    })
    .then(() => {
      if (owner !== accountGeneration) return;
      getSessions();
      if (currentSessionId.value === sessionId) {
        if (chatMode.value === 'chat') {
          currentSessionId.value = null;
          messageList.value = [];
          sessionDetailError.value = '';
        } else {
          clearChat();
        }
      }
      proxy.$modal.msgSuccess("删除成功");
    })
    .catch(() => {});
}

function handleDeleteMessage(message) {
  if (chatMode.value !== 'chat' || !message?.id || loading.value) return;
  const owner = accountGeneration, sessionId = currentSessionId.value;
  proxy.$modal.confirm('是否确认删除这条消息？')
    .then(() => {
      if (owner !== accountGeneration || sessionId !== currentSessionId.value) throw new Error('当前会话已变化');
      return deleteRagflowChatMessage(sessionId, message.id);
    })
    .then(({ data }) => {
      if (owner !== accountGeneration || sessionId !== currentSessionId.value) return;
      messageList.value = data?.messages || [];
      proxy.$modal.msgSuccess('消息已删除');
    })
    .catch(error => {
      if (owner === accountGeneration && !['cancel', 'close'].includes(error)) {
        proxy.$modal.msgError(error?.response?.data?.detail || '删除消息失败，请刷新后重试');
      }
    });
}

async function sendRequest(text, images, retrySupplementFrom = null, conditions = undefined) {
  const epoch = ++chatGeneration, isProduct = chatMode.value === 'product';
  if (isProduct && !currentModelId.value) {
    proxy.$modal.msgError("请先选择模型");
    return;
  }

  loading.value = true;
  const imageList = images ? images.slice() : [];

  const aiMsgIndex =
    messageList.value.push({
      role: "assistant",
      content: "",
      reasoningContent: "",
    }) - 1;
  scrollToBottom();
  isAutoScroll.value = true;

  const controller = new AbortController();
  abortController.value = controller;
  const targetMessage = messageList.value[aiMsgIndex];
  if (isProduct) Object.assign(targetMessage, { mode: 'product', productStatus: 'running', sessionId: currentSessionId.value });
  if (isProduct) {
    currentSessionId.value ||= uuidv4();
    Object.assign(targetMessage, { sessionId: currentSessionId.value, generationId: uuidv4(), messageId: uuidv4(), retrievalOnly: true, requestMessage: text, requestImages: imageList, requestConditions: conditions ?? null, libraryIds: [...productLibraryIds.value] });
  }
  const requestBody = isProduct ? { retrievalOnly: !retrySupplementFrom, generationId: targetMessage.generationId, messageId: targetMessage.messageId, ...(retrySupplementFrom ? { retrySupplementFrom } : {}), mode: 'product', ...(conditions === undefined ? {} : { conditions }), images: imageList, isReasoning: chatConfig.isReasoning, modelId: currentModelId.value, message: text, sessionId: currentSessionId.value, libraryIds: [...productLibraryIds.value], productId: selectedProductId.value } : {
    message: text, sessionId: currentSessionId.value,
  };
  const endpoint = isProduct ? '/ai/chat/send' : '/ai/chat/ragflow/stream';

  try {
    const response = await fetch(
      import.meta.env.VITE_APP_BASE_API + endpoint,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: "Bearer " + getToken(),
        },
        signal: controller.signal,
        body: JSON.stringify(requestBody),
      },
    );

    if (!response.ok || !response.body) {
      const errorText = await response.text().catch(() => "");
      if (isProduct) {
        let body; try { body = JSON.parse(errorText); } catch { body = {}; }
        throw Object.assign(new Error(body.userMessage || body.msg || '产品查询请求失败，请检查权限及输入'), { status: response.status });
      }
      let detail = errorText;
      try { detail = JSON.parse(errorText).detail || errorText; } catch { /* Keep the raw response. */ }
      throw new Error(detail || response.statusText || `HTTP ${response.status}`);
    }

    const reader = response.body.getReader();
    const decoder = new TextDecoder();
    let aiContent = "";
    let aiReasoning = "";
    let buffer = "";
    let needRefreshSessions = false;
    function acceptProductEvent(data) {
      if (!applyProductEvent(targetMessage, data)) return;
      currentSessionId.value = data.sessionId;
      if (data.type === 'meta') needRefreshSessions = true;
      if (data.type === 'run_info') currentRunId.value = data.runId;
      if (data.type === 'parameter_result') selectedProductId.value = data.productStatus === 'CONFIRMED' ? data.productId : null;
    }

    while (true) {
      if (controller.signal.aborted || epoch !== chatGeneration) return;
      const { done, value } = await reader.read();
      if (epoch !== chatGeneration || controller.signal.aborted) return;
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split("\n");
      buffer = lines.pop();

      for (const line of lines) {
        if (!line.trim()) continue;
        try {
          const data = JSON.parse(line);
          if (isProduct) {
            acceptProductEvent(data);
            continue;
          }
          if (data.type === "content") {
            aiContent += data.content;
            messageList.value[aiMsgIndex].content = aiContent;
          } else if (data.type === "reasoning") {
            aiReasoning += data.content;
            messageList.value[aiMsgIndex].reasoningContent = aiReasoning;
          } else if (data.type === "meta") {
            currentSessionId.value = data.session_id;
            if (!sessionList.value.find((s) => s.sessionId === data.session_id)) {
              needRefreshSessions = true;
            }
          } else if (data.type === "run_info") {
            currentRunId.value = data.run_id;
          } else if (data.type === "metrics") {
            messageList.value[aiMsgIndex].metrics = data.metrics;
          } else if (data.type === "error") {
            const errorMessage = data.error || "模型调用失败";
            aiContent = errorMessage;
            messageList.value[aiMsgIndex].content = errorMessage;
            proxy.$modal.msgError(errorMessage);
          }
        } catch (e) {
          if (isProduct) throw e;
          console.error("Parse error", e);
        }
      }
    }

    if (isProduct) {
      buffer += decoder.decode();
      if (buffer.trim()) acceptProductEvent(JSON.parse(buffer));
      if (!['completed', 'stopped', 'failed'].includes(targetMessage.productStatus)) targetMessage.productStatus = 'interrupted';
    } else {
      try {
        const sessionId = currentSessionId.value;
        const { data } = await getRagflowChatSession(sessionId);
        if (epoch === chatGeneration && sessionId === currentSessionId.value) {
          messageList.value = data.messages || [];
        }
      } catch { /* Keep the streamed text until the next explicit refresh. */ }
    }
    if (needRefreshSessions) {
      getSessions();
    }
  } catch (err) {
    if (epoch !== chatGeneration) return;
    if (isProduct) {
      if (['completed', 'stopped', 'failed'].includes(targetMessage.productStatus)) return;
      targetMessage.productStatus = err.status ? 'failed' : 'interrupted';
      targetMessage.productErrors = [...(targetMessage.productErrors || []), err.status ? err.message : '连接已结束，请确认本轮结果；已返回内容保留。'];
    } else if (err.name === "AbortError") {
      // 用户终止
    } else {
      const errorMessage = "请求失败: " + err.message;
      messageList.value[aiMsgIndex].content = errorMessage;
      proxy.$modal.msgError(errorMessage);
    }
  } finally {
    if (epoch === chatGeneration && abortController.value === controller) { loading.value = false; abortController.value = null; }
  }
}

function clearChat() {
  restoreQueryConditions(null);
  sessionDetailError.value = '';
  invalidateStream(); selectedProductId.value = null;
  messageList.value = [];
  currentSessionId.value = chatMode.value === 'product' ? generateSessionId() : null;
  currentSessionAgentData.value = null;
}

async function copyText(text) {
  if (!text) {
    proxy.$modal.msgWarning("内容为空，无法复制");
    return;
  }
  try {
    await navigator.clipboard.writeText(text);
    proxy.$modal.msgSuccess("复制成功");
  } catch {
    proxy.$modal.msgError("复制失败，请选择内容手动复制");
  }
}

function triggerImageUpload() {
  if (userConfig.visionEnabled !== '0' || loading.value) return;
  const input = imageInputRef.value;
  if (input) {
    input.value = "";
    input.click();
  }
}

let imageUploadGeneration = 0;
watch(() => userConfig.visionEnabled, () => {
  imageUploadGeneration++;
  if (userConfig.visionEnabled !== '0') inputImages.value = [];
}, { flush: 'sync' });

async function handleImageInputChange(event) {
  if (userConfig.visionEnabled !== '0' || loading.value) return;
  const owner = accountGeneration, epoch = chatGeneration, upload = ++imageUploadGeneration;
  const isCurrent = () => owner === accountGeneration && epoch === chatGeneration && upload === imageUploadGeneration && userConfig.visionEnabled === '0';
  const files = Array.from(event.target.files || []);
  if (!files.length) return;
  if (files.length + inputImages.value.length > 10) {
    proxy.$modal.msgError("最多只能上传 10 张图片");
    return;
  }
  const maxImageMb = Math.min(userConfig.imageMaxSizeMb || 5, 10);
  const maxSize = maxImageMb * 1024 * 1024;
  for (const file of files) {
    if (file.size > maxSize) {
      proxy.$modal.msgError(`单张图片大小不能超过 ${maxImageMb} MB`);
      return;
    }
  }
  try {
    proxy.$modal.loading("正在上传图片，请稍候...");
    for (const file of files) {
      if (!isCurrent()) return;
      const form = new FormData();
      form.append("file", file);
      const resp = await fetch(
        import.meta.env.VITE_APP_BASE_API + "/ai/chat/images",
        {
          method: "POST",
          headers: { Authorization: "Bearer " + getToken() },
          body: form,
        },
      );
      const data = await resp.json();
      if (!isCurrent()) return;
      if (resp.ok && data.code === 200 && data.data?.imageRef) {
        inputImages.value.push(data.data.imageRef);
      } else {
        proxy.$modal.msgError(data.msg || "上传图片失败");
      }
    }
  } catch (e) {
    if (isCurrent()) proxy.$modal.msgError("上传图片失败");
  } finally {
    if (owner === accountGeneration) proxy.$modal.closeLoading();
  }
}

async function handleSend(event, retrySupplementFrom = null) {
  if (chatMode.value === 'product' && productLibrariesLoading.value) { proxy.$modal.msgError('正在核对资料库范围，请稍候'); return; }
  if (sessionDetailError.value) { proxy.$modal.msgError('请先重新读取当前会话，或新建对话'); return; }
  if (loading.value || event?.isComposing) return;
  const text = inputMessage.value.trim();
  const images = chatMode.value === 'product' ? inputImages.value : [];
  if (chatMode.value === 'product' && images.length && userConfig.visionEnabled !== '0') { proxy.$modal.msgError('视觉功能已关闭，请移除图片后发送'); return; }
  if (!text && !images.length) return;
  if (chatMode.value === 'product' && !currentModelId.value) {
    proxy.$modal.msgError("请先选择模型");
    return;
  }

  if (chatMode.value === 'chat') {
    if (ragflowKnowledgeBasesLoading.value || ragflowKnowledgeBasesSaving.value) {
      proxy.$modal.msgWarning('知识库范围正在更新，请稍候');
      return;
    }
    try { await ensureRagflowSession(); }
    catch (error) {
      proxy.$modal.msgError(error?.response?.data?.detail || '新建对话失败，请稍后重试');
      return;
    }
  }

  if (chatMode.value === 'product' && (!productLibraryIds.value.length || !text)) {
    proxy.$modal.msgError('产品模式需要选择资料库并输入本轮要查询或核对的问题'); return;
  }
  let conditions;
  try { if (chatMode.value === 'product') conditions = queryConditionsPayload(); }
  catch (error) { proxy.$modal.msgError(error.message); return; }
  const imageList = images.slice();
  messageList.value.push({ role: "user", content: text, images: imageList });
  inputMessage.value = "";
  inputImages.value = [];
  currentRunId.value = null;

  await sendRequest(text, imageList, retrySupplementFrom, conditions);
}

function stopGeneration() {
  const message = messageList.value[messageList.value.length - 1];
  if (message?.mode === 'product' && !['completed', 'stopped', 'failed'].includes(message.productStatus)) message.productStatus = 'interrupted';
  if (abortController.value) {
    const controller = abortController.value;
    abortController.value = null;
    loading.value = false;

    const runId = currentRunId.value || (message?.mode === 'product' ? message.generationId : null);
    if (runId) {
      cancelChatRun(runId, message?.mode === 'product' ? message.sessionId : undefined, message?.mode === 'product' ? message.generationId : undefined)
        .then(() => { if (message?.mode === 'product' && messageList.value.includes(message)) return recoverProduct(message); })
        .catch((err) => {
          console.error("Failed to cancel run:", err);
          if (message?.mode === 'product' && messageList.value.includes(message)) message.productErrors = ['停止状态尚未确认，请使用“确认 / 恢复本轮结果”核对后再重试。'];
        })
        .finally(() => {
          controller.abort();
        });
    } else {
      controller.abort();
    }
  }
}

function handleScroll(e) {
  if (isProgrammaticScroll.value) return;
  const { scrollTop, scrollHeight, clientHeight } = e.target;
  const distanceToBottom = scrollHeight - scrollTop - clientHeight;
  if (distanceToBottom > 100) {
    isAutoScroll.value = false;
  } else if (distanceToBottom < 20) {
    isAutoScroll.value = true;
  }
}

function scrollToBottom() {
  if (isAutoScroll.value && chatHistoryRef.value) {
    isProgrammaticScroll.value = true;
    chatHistoryRef.value.scrollTop = chatHistoryRef.value.scrollHeight;
    requestAnimationFrame(() => {
      if (chatHistoryRef.value && isAutoScroll.value) {
        chatHistoryRef.value.scrollTop = chatHistoryRef.value.scrollHeight;
      }
    });
    if (scrollTimeout) clearTimeout(scrollTimeout);
    scrollTimeout = setTimeout(() => {
      isProgrammaticScroll.value = false;
      scrollTimeout = null;
    }, 100);
  }
}

function handleMainAction() {
  if (loading.value) {
    stopGeneration();
  } else {
    handleSend();
  }
}

useResizeObserver(chatContentRef, () => {
  if (isAutoScroll.value) {
    scrollToBottom();
  }
});

function clearAccountState() {
  accountGeneration++; clearChat(); ElMessageBox.close();
  showWelcome.value = true; startingChat.value = false; ragflowReady.value = false;
  ragflowKnowledgeBases.value = []; ragflowLibraryIds.value = [];
  ragflowKnowledgeBasesLoading.value = false; ragflowKnowledgeBasesSaving.value = false;
  ragflowKnowledgeBasesError.value = ''; ragflowKnowledgeBasesRequest++;
  sessionSearch.value = ''; renamingSessionId.value = null; sidebarCollapsed.value = false;
  inputMessage.value = ''; inputImages.value = []; sessionList.value = []; sessionLoading.value = false; sessionError.value = ''; sessionRequest++;
  productLibraryIds.value = []; productLibraries.value = []; productLibrariesError.value = ''; chatMode.value = 'chat';
  modelOptions.value = []; currentModelId.value = undefined; showConfigDialog.value = false;
  modelLoading.value = false; modelsLoaded.value = false; modelError.value = ''; modelRequest++;
  configRequest++; configLoading.value = false; configLoaded.value = false; configSaving.value = false; configError.value = '';
  Object.assign(userConfig, defaultUserConfig); Object.assign(editingUserConfig, defaultUserConfig);
  chatConfig.temperature = undefined; chatConfig.isReasoning = true;
  if (imageInputRef.value) imageInputRef.value.value = '';
  proxy.$modal.closeLoading();
}
watch(() => [account.id, account.token], () => {
  clearAccountState();
  if (account.id && account.token) { getModels(); getSessions(); loadUserConfig(); }
}, { flush: 'sync' });
onMounted(() => {
  getModels();
  getSessions();
  loadUserConfig();
});
</script>

<style scoped lang="scss">
/* 欢迎页样式 */
.welcome-page {
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--el-bg-color-page);
}
.welcome-page.is-starting {
  pointer-events: none;
}
.start-chat-btn {
  padding: 16px 48px;
  font-size: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.1);
  transition: all 0.3s;
}
.start-chat-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.15);
}

.product-query-conditions { margin: 0 20px; }
.query-condition-row { display: flex; gap: 8px; flex-wrap: wrap; margin: 8px 0; }
.query-condition-row .el-input, .query-condition-row .el-select { width: 200px; }
.selected-image-wrapper { display: flex; flex-direction: column; align-items: center; gap: 4px; }
@media (min-width: 1200px) {
  .chat-container.has-evidence-panel .chat-main { margin-right: min(48vw, 840px); }
}

.session-load-error { padding: 12px; overflow-wrap: anywhere; }
.product-scope { display: flex; flex-wrap: wrap; align-items: center; gap: 12px; padding: 12px; }

.chat-container {
  height: calc(100vh - 84px);
  padding: 0;
  background-color: var(--el-bg-color-page);
  overflow: hidden;
}

/* ===== 侧边栏美化 ===== */
.session-sidebar {
  width: 340px !important;
  min-width: 340px;
  flex: 0 0 340px;
  box-sizing: border-box;
  border-right: 1px solid var(--el-border-color-lighter);
  background-color: var(--el-bg-color); /* 适配夜间模式 */
  display: flex;
  flex-direction: column;
  z-index: 10;
  margin-bottom: 0;
  overflow: hidden;

  /* 顶部操作区 */
  .sidebar-top-actions {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 12px 4px;

    .mode-select {
      width: 110px;
      :deep(.el-input__wrapper) {
        box-shadow: none !important;
        background-color: transparent;
        padding-left: 4px;
        .el-input__inner {
          font-weight: 600;
          color: var(--el-text-color-primary);
        }
      }
    }

    .top-icons {
      display: flex;
      gap: 8px;
      .icon-btn {
        color: var(--el-text-color-regular);
        padding: 4px;
        &:hover { color: var(--el-color-primary); }
      }
    }
  }

  .sidebar-header {
    padding: 12px 14px 10px;

    .new-chat-btn {
      width: 100%;
      height: 46px;
      border-radius: 14px;
      font-size: 15px;
      font-weight: 600;
      letter-spacing: 0.2px;
      color: var(--el-color-primary);
      background: linear-gradient(135deg, var(--el-color-primary-light-9), var(--el-fill-color-light));
      border: 1px solid var(--el-border-color-lighter);
      box-shadow: 0 3px 10px rgba(64, 158, 255, 0.08);
      transition: transform 0.18s ease, box-shadow 0.18s ease, border-color 0.18s ease;
      display: flex;
      justify-content: center;
      align-items: center;

      &:hover {
        border-color: var(--el-color-primary-light-5);
        background: var(--el-color-primary-light-9);
        box-shadow: 0 5px 14px rgba(64, 158, 255, 0.14);
        transform: translateY(-1px);
      }
      &:active { transform: translateY(0); }
      .btn-icon { margin-right: 7px; font-size: 17px; }
    }
  }

  .sidebar-search {
    width: auto;
    min-width: 0;
    margin: 0 14px 12px;
    padding: 9px;
    box-sizing: border-box;
    border: 1px solid var(--el-border-color-lighter);
    border-radius: 14px;
    background: var(--el-fill-color-lighter);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.025);

    :deep(.el-input__wrapper) {
      min-height: 38px;
      padding: 0 11px;
      border-radius: 9px;
      background: var(--el-bg-color);
      box-shadow: 0 0 0 1px var(--el-border-color-light) inset;
      transition: box-shadow 0.2s ease;
    }
    :deep(.el-input__wrapper.is-focus) {
      box-shadow: 0 0 0 1px var(--el-color-primary) inset;
    }
    :deep(.el-input__inner) { font-size: 14px; }
  }

  .session-list {
    flex: 1;
    min-width: 0;
    width: 100%;
    box-sizing: border-box;
    overflow-y: auto;
    padding: 4px 10px;
    &::-webkit-scrollbar { width: 4px; }
    &::-webkit-scrollbar-thumb { background: var(--el-border-color); border-radius: 2px; }

    .session-group-label {
      font-size: 12px;
      color: var(--el-text-color-secondary);
      padding: 8px 8px 4px;
      font-weight: 500;
    }

    .session-item {
      display: flex;
      align-items: center;
      padding: 11px 12px;
      margin-bottom: 5px;
      background-color: transparent;
      border-radius: 10px;
      min-height: 44px;
      cursor: pointer;
      transition: all 0.2s;
      position: relative;
      border: none;

      &:hover {
        background-color: var(--el-fill-color-light); /* 适配夜间模式 */
        .session-actions { opacity: 1; }
      }
      &.active {
        background-color: var(--el-color-primary-light-9); /* 适配夜间模式 */
        .session-title { color: var(--el-color-primary); font-weight: 500; }
      }

      .session-open {
        display: flex; align-items: center; flex: 1; min-width: 0; border: 0; padding: 0; background: transparent; text-align: left; font: inherit; cursor: pointer;
        &:focus-visible { outline: 2px solid var(--el-color-primary); outline-offset: 2px; }
        .session-title {
          display: -webkit-box;
          -webkit-box-orient: vertical;
          -webkit-line-clamp: 2;
          overflow: hidden;
          overflow-wrap: anywhere;
          line-height: 1.5;
          max-height: 3em;
          font-size: 14px;
          color: var(--el-text-color-primary);
        }
      }

      .session-actions {
        display: flex;
        align-items: center;
        gap: 2px;
        opacity: 0;
        transition: opacity 0.2s;
        margin-left: 8px;
        :deep(.el-button) {
          padding: 4px;
          margin: 0;
          height: auto;
        }
      }
    }
    .empty-session { text-align: center; color: var(--el-text-color-secondary); font-size: 13px; margin-top: 40px; }
  }
}

.chat-main {
  padding: 0;
  height: 100%;
  display: flex;
  flex-direction: column;
  background-color: var(--el-bg-color-page);
  position: relative;
  overflow: hidden;

  .chat-header {
    height: 60px;
    background-color: var(--el-bg-color);
    border-bottom: 1px solid var(--el-border-color-lighter);
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 20px;
    .header-title { font-size: 16px; font-weight: 600; color: var(--el-text-color-primary); margin-left: 8px; }

    .expand-btn {
      color: var(--el-text-color-regular);
      &:hover { color: var(--el-color-primary); }
    }

    .header-right-actions {
      display: flex;
      align-items: center;
      gap: 16px;
      .product-header-actions { display: flex; align-items: center; }
    }
  }

  .chat-history {
    flex: 1;
    overflow-y: auto;
    padding: 20px;
    .chat-content {
      min-height: 100%;
      padding-bottom: 20px;
      &.is-empty { display: flex; flex-direction: column; height: 100%; }
    }

    .welcome-screen {
      flex: 1;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      color: var(--el-text-color-secondary);
      opacity: 0.8;
      .welcome-icon { background: var(--el-fill-color); border-radius: 50%; padding: 20px; margin-bottom: 20px; color: var(--el-color-primary); }
      h2 { margin-bottom: 10px; font-weight: 500; }
    }

    .message-row {
      display: flex;
      max-width: 900px;
      margin-bottom: 24px;
      margin-left: auto;
      margin-right: auto;

      .message-avatar {
        flex-shrink: 0;
        margin-right: 12px;
        margin-top: 2px;
        .avatar-user { background-color: var(--el-color-primary); }
        .avatar-ai { background-color: var(--el-color-success); }
      }

      .message-content-wrapper {
        flex: 1;
        display: flex;
        flex-direction: column;
        max-width: calc(100% - 52px);

        .message-sender {
          font-size: 12px;
          color: var(--el-text-color-secondary);
          margin-bottom: 4px;
          display: flex;
          align-items: center;
          gap: 10px;
        }
        .message-time { font-size: 11px; opacity: 0.8; }
        .message-bubble {
          padding: 12px 16px;
          border-radius: 12px;
          font-size: 15px;
          line-height: 1.6;
          max-width: 100%;
          min-width: 60px;
        }
        .ragflow-evidence { display: flex; flex-wrap: wrap; gap: 10px; margin-top: 12px; }
        .ragflow-evidence-item { width: 180px; margin: 0; border: 1px solid var(--el-border-color-light); border-radius: 8px; overflow: hidden; background: var(--el-fill-color-lighter); }
        .ragflow-evidence-image { display: block; width: 180px; height: 120px; cursor: zoom-in; }
        .ragflow-evidence-item figcaption { display: flex; flex-direction: column; gap: 2px; padding: 6px 8px; font-size: 12px; color: var(--el-text-color-secondary); }
        .ragflow-evidence-excerpt { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

        .message-footer {
          margin-top: 6px;
          display: flex;
          justify-content: space-between;
          align-items: center;
          width: 100%;
          .message-metrics { font-size: 12px; color: var(--el-text-color-secondary); display: flex; flex-wrap: wrap; gap: 8px; }
          .footer-actions { display: flex; align-items: center; gap: 10px; }
          .model-info { margin-left: auto; }
        }
      }

      &.message-user {
        flex-direction: row-reverse;
        padding-left: 52px;
        .message-avatar { margin-left: 12px; margin-right: 0; }
        .message-content-wrapper {
          align-items: flex-end;
          .message-sender { flex-direction: row-reverse; }
          .message-bubble {
            background-color: var(--el-color-primary);
            color: #fff;
            border-top-right-radius: 2px;
            .user-text { white-space: pre-wrap; word-break: break-word; }
            .user-images {
              display: flex;
              flex-wrap: wrap;
              gap: 8px;
              margin-bottom: 8px;
              justify-content: flex-end;
              .user-image-item { width: 100px; height: 100px; border-radius: 4px; cursor: pointer; background-color: var(--app-surface-overlay, rgba(255, 255, 255, 0.1)); }
            }
          }
          .message-footer {
            justify-content: flex-end;
            .footer-actions { flex-direction: row-reverse; }
          }
        }
      }

      &.message-ai {
        padding-right: 52px;
        .message-content-wrapper {
          align-items: stretch;
          .message-bubble { background-color: var(--el-bg-color); border: 1px solid var(--el-border-color); border-top-left-radius: 2px; box-shadow: 0 2px 6px rgba(0, 0, 0, 0.02); }
        }
      }
    }
  }

  /* 输入区域外观 */
  .chat-input-area {
    background-color: var(--el-bg-color-page);
    padding: 10px 20px 20px;

    .input-wrapper {
      max-width: 900px;
      margin: 0 auto;
      border: 1px solid var(--el-border-color);
      border-radius: 16px;
      padding: 12px 16px;
      background-color: var(--el-bg-color);
      box-shadow: 0 4px 16px rgba(0, 0, 0, 0.04);
      transition: all 0.3s;
      display: flex;
      flex-direction: column;
      gap: 8px;

      &:focus-within {
        border-color: var(--el-color-primary);
        box-shadow: 0 4px 20px var(--el-color-primary-light-9);
      }

      :deep(.el-textarea__inner) {
        border: none;
        box-shadow: none !important;
        padding: 0;
        resize: none;
        max-height: 200px;
        background-color: transparent;
        color: var(--el-text-color-primary);
        font-size: 15px;
        line-height: 1.5;
      }

      .selected-knowledge-bases {
        display: flex;
        flex-wrap: wrap;
        gap: 8px;
        margin-top: 4px;

        .kb-pill {
          display: inline-flex;
          align-items: center;
          background-color: var(--el-fill-color-light);
          color: var(--el-text-color-regular);
          border-radius: 8px;
          padding: 4px 10px;
          font-size: 13px;
          line-height: 1;
          border: 1px solid var(--el-border-color-lighter);
          transition: background 0.2s;

          .kb-name { margin-right: 6px; }
          .kb-close {
            cursor: pointer;
            font-size: 14px;
            opacity: 0.6;
            transition: opacity 0.2s;
            &:hover { opacity: 1; color: var(--el-color-danger); }
          }
        }
      }

      .selected-images {
        display: flex;
        flex-wrap: wrap;
        gap: 8px;
        margin-top: 4px;
        .selected-image-item { width: 60px; height: 60px; border-radius: 6px; cursor: pointer; border: 1px solid var(--el-border-color); background-color: var(--el-fill-color-light); }
      }

      .input-actions {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-top: 4px;

        .left-actions {
          display: flex;
          align-items: center;
          gap: 10px;

          .add-kb-btn {
            border-radius: 6px;
            color: var(--el-text-color-regular);
            border-color: var(--el-border-color);
            &:hover { color: var(--el-color-primary); border-color: var(--el-color-primary); }
          }

          .toggle-chip { border-radius: 999px; margin-left: 0; }
        }

        .send-btn {
          padding: 10px;
          font-size: 18px;
          transition: transform 0.2s;
          &:hover:not(:disabled) { transform: scale(1.05); }
        }
      }
    }
  }
}

/* 弹窗重新设计 */
.search-dialog {
  .search-hint {
    margin-top: 12px;
    font-size: 13px;
    color: var(--el-text-color-secondary);
  }
}

.kb-dialog {
  :deep(.el-dialog__header) { margin-bottom: 16px; }
  :deep(.el-dialog__body) { padding-top: 0; }

  .kb-dialog-content {
    min-height: 200px;
  }
  .kb-checkbox-list {
    max-height: 400px;
    overflow-y: auto;
    padding-right: 8px;

    &::-webkit-scrollbar { width: 4px; }
    &::-webkit-scrollbar-thumb { background: var(--el-border-color); border-radius: 2px; }

    .kb-checkbox-item {
      display: flex;
      width: 100%;
      margin-right: 0;
      margin-bottom: 8px;
      padding: 12px;
      border: 1px solid var(--el-border-color-lighter);
      border-radius: 8px;
      transition: all 0.2s;

      &:hover { background-color: var(--el-fill-color-light); }
      &:last-child { margin-bottom: 0; }

      :deep(.el-checkbox__label) {
        display: flex;
        flex-direction: column;
        white-space: normal;
        line-height: 1.4;
      }

      .kb-item-name { font-size: 14px; color: var(--el-text-color-primary); font-weight: 500; }
      .kb-item-desc { font-size: 12px; color: var(--el-text-color-secondary); margin-top: 4px; }
    }
  }
}

.chat-config-dialog {
  :deep(.el-dialog__body) { padding-top: 10px; padding-bottom: 10px; }
  :deep(.el-form-item) { margin-bottom: 16px; }
  :deep(.el-form-item__label) { font-size: 13px; color: var(--el-text-color-secondary); }
}
</style>
