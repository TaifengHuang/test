<template>
  <div class="app-container chat-container" :class="{ 'has-evidence-panel': activeSourceMessage }">
    <!-- 每次进入都显示的欢迎页 -->
    <div v-if="showWelcome" class="welcome-page" v-loading="startingChat || checkingRagflowChat"
         :class="{ 'is-starting': startingChat || checkingRagflowChat }">
      <el-button v-if="!checkingRagflowChat" type="primary" size="large" class="start-chat-btn" @click="startChat">
        开始聊天
      </el-button>
    </div>

    <!-- 会话界面 -->
    <el-container v-else style="height: 100%">
      <!-- 侧边栏：会话历史 -->
      <el-aside v-show="!sidebarCollapsed && !activeSourceMessage" width="340px" class="session-sidebar">
        <!-- 侧边栏顶部操作区 -->
        <div class="sidebar-top-actions">
          <!-- 聊天模式下拉选择 -->
          <el-select v-model="chatMode" :disabled="loading" aria-label="聊天模式" class="mode-select"
                     @change="changeChatMode" title="切换模式会新建会话">
            <el-option value="chat" label="普通聊天"/>
            <el-option value="product" label="参数问答"/>
          </el-select>

          <div class="top-icons">
            <!-- 收起侧边栏图标 -->
            <el-tooltip content="收起会话栏" placement="bottom">
              <el-button link class="icon-btn" @click="toggleSessionSidebar">
                <SidebarCollapseIcon size="18"/>
              </el-button>
            </el-tooltip>
          </div>
        </div>

        <!-- 新建对话按钮 -->
        <div class="sidebar-header">
          <el-button class="new-chat-btn" @click="newConversation">
            <el-icon class="btn-icon">
              <Plus/>
            </el-icon>
            新建对话
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
        <!-- 搜索框与会话列表之间的短分隔线 -->
        <div class="session-search-divider" aria-hidden="true"></div>

        <!-- 会话列表 -->
        <div class="session-list" v-loading="sessionLoading">
          <div v-if="filteredSessions.length > 0" class="session-group-label">最近</div>

          <div v-for="session in filteredSessions" :key="session.sessionId"
               :class="['session-item', currentSessionId === session.sessionId ? 'active' : '']">
            <button type="button" class="session-open" :aria-label="`打开会话：${session.sessionTitle || '新对话'}`"
                    :aria-pressed="currentSessionId === session.sessionId" @click="loadSession(session.sessionId)">
              <span class="session-title">{{ session.sessionTitle || "新对话" }}</span>
            </button>
            <div class="session-actions">
              <el-button link icon="Edit" :aria-label="`重命名`" :disabled="!!renamingSessionId"
                         @click.stop="handleRenameSession(session)"/>
              <el-button class="delete-btn" type="danger" link icon="Delete" :aria-label="`删除`"
                         @click.stop="handleDeleteSession(session.sessionId)"/>
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
              <SidebarCollapseIcon size="18"/>
            </el-button>
            <span class="header-title">{{ currentSessionTitle }}</span>
          </div>
          <div class="header-right-actions">
            <el-button class="logout-btn" plain @click="handleLogout">
              退出登录
            </el-button>
            <div v-if="chatMode === 'product'" class="product-header-actions">
              <el-tooltip content="全局参数配置" placement="bottom">
                <el-button icon="Setting" aria-label="全局参数配置" circle style="margin-right: 10px"
                           @click="openConfigDialog"></el-button>
              </el-tooltip>
              <el-select v-model="currentModelId" placeholder="选择模型" aria-label="选择聊天模型"
                         :loading="modelLoading" :disabled="modelLoading || loading" size="large" style="width: 210px">
                <el-option v-for="item in modelOptions" :key="item.modelId"
                           :label="`${item.provider}/${item.modelCode}`" :value="item.modelId"/>
              </el-select>
            </div>
          </div>
        </div>

        <div v-if="chatMode === 'product' && (modelError || (modelsLoaded && !modelOptions.length))" role="status"
             class="session-load-error">
          <p>{{ modelError || '当前没有可选择的模型，请联系管理员配置后刷新。' }}</p>
          <el-button :loading="modelLoading" @click="getModels">重新读取模型</el-button>
        </div>
        <div v-if="chatMode === 'product'" class="product-scope">
          <el-select v-model="BidKnowledgeLibraryIds" multiple filterable :disabled="loading || productLibrariesLoading"
                     placeholder="选择产品知识库" aria-label="选择产品知识库" style="min-width:260px"
                     @change="selectedProductId = null">
            <el-option v-for="library in productLibraries" :key="library.libraryId" :value="library.libraryId"
                       :label="library.name" :disabled="library.status !== 'ACTIVE'"/>
          </el-select>
          <el-button :loading="productLibrariesLoading" :disabled="loading" @click="loadProductLibraries">
            刷新产品知识库
          </el-button>
          <span v-if="productLibrariesError" role="status">{{ productLibrariesError }}</span>
          <span>检索所选授权产品知识库并结合知识库返回结果统一回答</span>
          <el-button v-if="selectedProductId" :disabled="loading" @click="selectedProductId = null">清除产品
            {{ selectedProductId }}
          </el-button>
        </div>
        <el-collapse v-if="chatMode === 'product'" class="product-query-conditions">
          <el-collapse-item title="适用条件（可选，对下一轮生效）" name="conditions">
            <el-select v-model="queryConditionMode" aria-label="本轮适用条件选择" :disabled="loading">
              <el-option value="any" label="未限定条件"/>
              <el-option value="none" label="明确无附加条件"/>
              <el-option value="specified" label="指定适用条件"/>
              <el-option v-if="queryHistoryConditions" value="history" label="沿用历史复合条件"/>
            </el-select>
            <p>问题涉及温度、配置等条件时，请在此明确填写；未填写时不会推断。</p>
            <p v-if="queryConditionMode === 'history'">{{
                conditionText(queryHistoryConditions)
              }}。重新指定或清除后可修改。</p>
            <template v-if="queryConditionMode === 'specified'">
              <div v-for="(row, index) in queryConditionRows" :key="index" class="query-condition-row">
                <el-input v-model="row.key" :disabled="loading" maxlength="100" placeholder="例如：环境温度"
                          :aria-label="`查询条件 ${index + 1} 名称`"/>
                <el-select v-model="row.type" :disabled="loading" :aria-label="`查询条件 ${index + 1} 类型`"
                           @change="row.value = row.type === 'boolean' ? null : ''">
                  <el-option value="text" label="文本 / 精确数值"/>
                  <el-option value="boolean" label="是 / 否"/>
                </el-select>
                <el-select v-if="row.type === 'boolean'" v-model="row.value" :disabled="loading"
                           :aria-label="`查询条件 ${index + 1} 值`">
                  <el-option :value="true" label="是"/>
                  <el-option :value="false" label="否"/>
                </el-select>
                <el-input v-else v-model="row.value" :disabled="loading" maxlength="500" placeholder="例如：40 ℃"
                          :aria-label="`查询条件 ${index + 1} 值`"/>
                <el-button :disabled="loading" @click="queryConditionRows.splice(index, 1)">移除</el-button>
              </div>
              <el-button :disabled="loading" @click="queryConditionRows.push({ key: '', type: 'text', value: '' })">
                添加条件
              </el-button>
            </template>
          </el-collapse-item>
        </el-collapse>
        <div v-if="sessionDetailError" role="status" class="session-load-error">
          <p>{{ sessionDetailError }}</p>
          <el-button :loading="loading" @click="loadSession(currentSessionId)">重试读取当前会话</el-button>
        </div>
        <div class="chat-history" ref="chatHistoryRef" @scroll="handleScroll">
          <div class="chat-content" ref="chatContentRef" :class="{ 'is-empty': messageList.length === 0 }">
            <div v-if="messageList.length === 0" class="welcome-screen">
              <div class="welcome-icon">
                <el-icon size="60">
                  <Service/>
                </el-icon>
              </div>
              <h2>你好！我是你的 AI 助手</h2>
              <p>请在下方输入问题开始对话...</p>
            </div>

            <div v-for="(msg, index) in messageList" :key="index"
                 :class="['message-row', msg.role === 'user' ? 'message-user' : 'message-ai']">
              <div class="message-avatar">
                <el-avatar :icon="msg.role === 'user' ? 'UserFilled' : 'Service'" :size="40"
                           :class="msg.role === 'user' ? 'avatar-user' : 'avatar-ai'"></el-avatar>
              </div>
              <div class="message-content-wrapper">
                <div class="message-sender">
                  {{ msg.role === "user" ? "我" : "AI 助手" }}
                  <span class="message-time" v-if="msg.createdAt">{{ formatTime(msg.createdAt) }}</span>
                </div>
                <div class="message-bubble">
                  <div v-if="msg.role === 'user'">
                    <div v-if="msg.images && msg.images.length > 0" class="user-images">
                      <ChatImage v-for="(img, idx) in msg.images" :key="img + ':' + idx" :image="img"
                                 :session-id="currentSessionId || ''" :message-id="msg.id || ''"
                                 class="user-image-item"/>
                    </div>
                    <div class="user-text">{{ msg.content }}</div>
                  </div>
                  <template v-else>
                    <ProductMessage v-if="msg.mode === 'product'" :message="msg"
                                    :source-active="activeSourceMessage === msg"
                                    @source-panel-change="setSourcePanel(msg, $event)" :loading="loading"
                                    :streaming="loading && index === messageList.length - 1"
                                    @choose="chooseProduct($event, msg)" @recover="recoverProduct(msg)"
                                    @stop="stopProductMessage(msg)" @retry="retryProduct(msg)"
                                    @retry-supplement="retryProduct(msg, true)" @access-lost="hideProductMessage(msg)"/>
                    <AiMessage v-else :content="msg.content" :reasoning-content="msg.reasoningContent"
                               :reasoning-status="msg.mode === 'product' ? msg.reasoningStatus : undefined"
                               :loading="loading && index === messageList.length - 1"/>
                  </template>
                  <div
                      v-if="msg.role === 'assistant' && msg.evidenceImages?.length"
                      class="ragflow-evidence"
                      @pointerdown="startEvidenceDrag"
                      @pointermove="moveEvidenceDrag"
                      @pointerup="endEvidenceDrag"
                      @pointercancel="endEvidenceDrag"
                      @click.capture="suppressEvidenceClick"
                      @dragstart.prevent
                  >
                    <figure v-for="(evidence, evidenceIndex) in msg.evidenceImages" :key="evidence.id"
                            class="ragflow-evidence-item">
                      <ChatImage :image="evidence.id" evidence :session-id="currentSessionId || ''"
                                 :message-id="msg.id || ''" class="ragflow-evidence-image"/>
                      <span class="ragflow-evidence-number">[{{ evidence.number ?? evidenceIndex + 1 }}]</span>
                      <button type="button" class="ragflow-evidence-file" :title="evidence.fileName || '打开所属文件'"
                              @click.stop="openEvidenceFile(evidence, msg)">
                        {{ evidence.fileName || '所属文件' }}
                      </button>
                    </figure>
                  </div>
                </div>
                <div class="message-footer">
                  <div class="footer-actions">
                    <!-- 删除按钮 -->
                    <el-tooltip v-if="chatMode === 'chat' && msg.id" content="删除消息" placement="top">
                      <el-button link type="danger" :icon="Delete" aria-label="删除消息" size="small"
                                 :disabled="loading" @click="handleDeleteMessage(msg)"></el-button>
                    </el-tooltip>

                    <!-- 复制按钮 -->
                    <el-tooltip content="复制" placement="top">
                      <el-button link type="info" :icon="DocumentCopy" aria-label="复制消息" size="small"
                                 @click="copyText(messageCopyText(msg))"></el-button>
                    </el-tooltip>

                    <!-- 耗时统计 -->
                    <div v-if="userConfig.metricsDefaultVisible == '0' && hasMetrics(msg)" class="message-metrics">
                      <span v-if="msg.metrics?.duration !== null && msg.metrics?.duration !== undefined">耗时 {{
                          msg.metrics.duration.toFixed(3)
                        }} s</span>
                      <span v-if="msg.metrics?.inputTokens !== null && msg.metrics?.inputTokens !== undefined">输入 {{
                          msg.metrics.inputTokens
                        }} tokens</span>
                      <span v-if="msg.metrics?.outputTokens !== null && msg.metrics?.outputTokens !== undefined">输出 {{
                          msg.metrics.outputTokens
                        }} tokens</span>
                      <span v-if="msg.metrics?.totalTokens !== null && msg.metrics?.totalTokens !== undefined">总 {{
                          msg.metrics.totalTokens
                        }} tokens</span>
                      <span v-if="msg.metrics?.reasoningTokens !== null && msg.metrics?.reasoningTokens !== undefined">推理 {{
                          msg.metrics.reasoningTokens
                        }} tokens</span>
                    </div>
                  </div>
                  <!-- 模型名称区域已移除 -->
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
<!--            <div v-if="chatMode === 'chat' && selectedKnowledgeBases.length > 0" class="selected-knowledge-bases">-->
<!--              <div class="kb-pill" v-for="kb in selectedKnowledgeBases" :key="kb.libraryId">-->
<!--                <span class="kb-name">{{ kb.name }}</span>-->
<!--                <el-icon class="kb-close" @click="removeKnowledgeBase(kb.libraryId)">-->
<!--                  <Close/>-->
<!--                </el-icon>-->
<!--              </div>-->
<!--            </div>-->

            <!-- 产品模式图片展示 -->
            <div class="selected-images"
                 v-if="chatMode === 'product' && userConfig.visionEnabled == '0' && inputImages.length">
              <div v-for="(img, idx) in inputImages" :key="img" class="selected-image-wrapper">
                <ChatImage :image="img" class="selected-image-item"/>
                <el-button link :disabled="loading" :aria-label="`移除本轮第 ${idx + 1} 张图片`"
                           @click="inputImages.splice(idx, 1)">移除
                </el-button>
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
                  选择知识库
                </el-button>

                <el-check-tag
                  v-if="chatMode === 'chat'"
                  :checked="isDeepThink"
                  @change="handleDeepThinkChange"
                  class="deep-think-tag"
                >
                  <!-- 选中时显示不同的图标，你可以根据项目里的 svg-icon 或 el-icon 替换 -->
                  <el-icon class="tag-icon" v-if="isDeepThink"><SuccessFilled /></el-icon>
                  <el-icon class="tag-icon" v-else><Connection /></el-icon>
                  <span>深度思考</span>
                </el-check-tag>

                <el-tooltip
                    v-if="chatMode === 'product' && currentModelInfo && currentModelInfo.supportImages === 'Y' && userConfig.visionEnabled == '0'"
                    content="上传图片" placement="top">
                  <el-button circle plain size="small" :icon="Picture" @click="triggerImageUpload"/>
                </el-tooltip>
                <el-button
                    v-if="chatMode === 'product' && currentModelInfo && currentModelInfo.supportReasoning === 'Y'"
                    class="toggle-chip" size="small" :type="chatConfig.isReasoning ? 'primary' : ''"
                    :plain="!chatConfig.isReasoning" @click="chatConfig.isReasoning = !chatConfig.isReasoning">
                  <template #icon>
                    <svg-icon icon-class="deepthink"/>
                  </template>
                  深度思考
                </el-button>
              </div>
              <el-button class="send-btn" :type="loading ? 'danger' : 'primary'"
                         :icon="loading ? 'VideoPause' : 'Promotion'" circle size="large" @click="handleMainAction"
                         :disabled="!loading && !inputMessage.trim() && !inputImages.length"/>
            </div>
          </div>
        </div>
      </el-main>
    </el-container>

    <!-- 搜索会话弹窗 -->
    <el-dialog v-model="searchDialogVisible" title="搜索历史对话" width="400px" append-to-body class="search-dialog"
               @opened="focusSearchInput">
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

    <!-- 知识库选择对话框 (按设计图美化) -->
    <el-dialog v-model="knowledgeBaseDialogVisible" width="540px" append-to-body class="kb-dialog" :show-close="true">
      <template #header>
        <div class="kb-dialog-header">
          <el-icon class="header-icon"><Coin /></el-icon>
          <span class="header-title">选择知识库</span>
        </div>
      </template>
      <div v-loading="ragflowKnowledgeBasesLoading" class="kb-dialog-content">
        <div class="kb-dialog-intro">
          <div class="kb-intro-mark">
            <span>库</span>
            <svg class="sparkle" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M12 2L14.5 9.5L22 12L14.5 14.5L12 22L9.5 14.5L2 12L9.5 9.5L12 2Z" fill="currentColor"/>
            </svg>
          </div>
          <div class="kb-intro-text">
            <strong>选择本次对话引用的知识库</strong>
            <p>仅包含已完成解析文件的知识库可以用于 AI 检索。</p>
          </div>
          <div class="kb-intro-illustration">
            <svg viewBox="0 0 48 48" fill="none" xmlns="http://www.w3.org/2000/svg">
              <rect x="8" y="6" width="26" height="32" rx="4" fill="#E1EFFF" opacity="0.6"/>
              <rect x="14" y="10" width="26" height="32" rx="4" fill="#D1E6FF" opacity="0.8"/>
              <rect x="20" y="14" width="20" height="24" rx="3" fill="#B8D8FF"/>
              <circle cx="32" cy="26" r="6" stroke="#409EFF" stroke-width="2" fill="white"/>
              <line x1="36.5" y1="30.5" x2="40" y2="34" stroke="#409EFF" stroke-width="2" stroke-linecap="round"/>
            </svg>
          </div>
        </div>

        <el-alert v-if="ragflowKnowledgeBasesError" :title="ragflowKnowledgeBasesError" type="error" show-icon
                  :closable="false" style="margin-bottom: 16px;"/>
        <el-alert
            v-if="unavailableKnowledgeBaseCount"
            :title="`${unavailableKnowledgeBaseCount} 个知识库暂无已解析完成的文件，已禁止选择`"
            type="warning"
            show-icon
            :closable="false"
            class="kb-unavailable-alert"
        />
        <div v-if="ragflowKnowledgeBases.length" class="kb-checkbox-list">
          <el-checkbox-group v-model="tempSelectedKnowledgeBaseIds">
            <el-checkbox
                v-for="(kb, index) in sortedRagflowKnowledgeBases"
                :key="kb.libraryId"
                :label="kb.libraryId"
                :disabled="kb.hasParsedFiles === false || ragflowKnowledgeBasesSaving"
                :class="['kb-checkbox-item', { 'is-unavailable': kb.hasParsedFiles === false }]"
            >
              <!-- 左侧图标 -->
              <div :class="['kb-item-icon', `icon-color-${index % 3}`, { 'is-disabled': kb.hasParsedFiles === false }]">
                <el-icon><Coin v-if="index % 3 === 0"/><Document v-else-if="index % 3 === 1"/><Tickets v-else/></el-icon>
              </div>

              <!-- 中间信息 -->
              <div class="kb-item-info">
                <div class="kb-item-title">
                  <span class="kb-item-name">{{ kb.name }}</span>
                  <span :class="['kb-status-tag', kb.hasParsedFiles === false ? 'tag-disabled' : 'tag-success']">
                    {{ kb.hasParsedFiles === false ? '暂不可引用' : '可引用' }}
                  </span>
                </div>
                <!-- 如果设计图不需要描述，可以注释掉下面这行，或者根据需求保留 -->
                <!-- <span class="kb-item-desc">{{ kb.hasParsedFiles === false ? '该知识库暂无已解析完成的文件，请先完成文件解析。' : (kb.description || '包含可检索的已解析文件') }}</span> -->
              </div>

              <!-- 右侧箭头 -->
              <div class="kb-item-arrow">
                <el-icon><ArrowRight /></el-icon>
              </div>
            </el-checkbox>
          </el-checkbox-group>
        </div>
        <el-empty v-else-if="!ragflowKnowledgeBasesLoading" description="暂无可用的知识库" :image-size="80"/>
      </div>
      <template #footer>
        <span class="dialog-footer">
          <span class="kb-selection-count">已选择 <span class="count-num">{{ tempSelectedKnowledgeBaseIds.length }}</span> 个</span>
          <div class="footer-btns">
            <el-button @click="knowledgeBaseDialogVisible = false">取消</el-button>
            <el-button type="primary" :loading="ragflowKnowledgeBasesSaving" :disabled="ragflowKnowledgeBasesLoading"
                       @click="confirmKnowledgeBaseSelection">确定</el-button>
          </div>
        </span>
      </template>
    </el-dialog>

    <!-- 全局配置弹窗 -->
    <el-dialog v-model="showConfigDialog" title="用户全局配置" width="700px" append-to-body class="chat-config-dialog">
      <p v-if="configError" role="alert">{{ configError }}</p>
      <el-button v-if="!configLoaded" :loading="configLoading" @click="loadUserConfig">重新读取配置</el-button>
      <el-form :model="editingUserConfig" label-width="150px"
               :disabled="!configLoaded || configLoading || configSaving">
        <el-row :gutter="20">
          <el-col :span="12">
            <el-form-item label="默认温度">
              <el-input-number v-model="editingUserConfig.temperature" :min="0" :max="2" :step="0.1" :precision="1"
                               placeholder="默认温度" style="width: 100%"/>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="附带历史消息">
              <el-switch active-value="0" inactive-value="1" v-model="editingUserConfig.addHistoryToContext"/>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="历史消息轮数" v-if="editingUserConfig.addHistoryToContext == '0'">
              <el-input-number v-model="editingUserConfig.numHistoryRuns" :min="1" :max="20" placeholder="历史消息轮数"
                               style="width: 100%"/>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="默认显示指标">
              <el-switch active-value="0" inactive-value="1" v-model="editingUserConfig.metricsDefaultVisible"/>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="开启视觉功能">
              <el-switch active-value="0" inactive-value="1" v-model="editingUserConfig.visionEnabled"/>
            </el-form-item>
          </el-col>
          <el-col :span="12">
            <el-form-item label="图片最大大小" v-if="editingUserConfig.visionEnabled === '0'">
              <el-input-number v-model="editingUserConfig.imageMaxSizeMb" :min="1" :max="50" placeholder="图片大小"
                               style="width: 100%">
                <template #suffix><span>MB</span></template>
              </el-input-number>
            </el-form-item>
          </el-col>
          <el-col :span="24">
            <el-form-item label="系统提示词">
              <el-input v-model="editingUserConfig.systemPrompt" type="textarea" :rows="4"
                        placeholder="设置全局系统提示词"/>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="showConfigDialog = false">取消</el-button>
          <el-button type="primary" :loading="configSaving" :disabled="!configLoaded || configLoading"
                     @click="handleSaveConfig">保存</el-button>
        </span>
      </template>
    </el-dialog>
    <input v-if="userConfig.visionEnabled === '0'" ref="imageInputRef" type="file" accept="image/*" multiple
           class="chat-image-input" @change="handleImageInputChange"/>
  </div>
</template>

<script setup name="AiChat">
import {ElMessageBox} from 'element-plus';
import {Plus, Close, Expand, Fold, Search, Coin, Document, Tickets, ArrowRight, Connection, SuccessFilled} from '@element-plus/icons-vue';
import useUserStore from '@/store/modules/user';
import {listModelAll} from "@/api/ai/model";
import {
  listChatSession, delChatSession, renameChatSession, getChatSession,
  getUserChatConfig, saveUserChatConfig, cancelChatRun, getProductGeneration,
  startRagflowChat, getRagflowChatStatus, getRagflowKnowledgeBases, updateRagflowKnowledgeBases,
  listRagflowChatSessions, createRagflowChatSession, getRagflowChatSession,
  renameRagflowChatSession, deleteRagflowChatSessions, deleteRagflowChatMessage,
  downloadRagflowEvidenceFile,
} from "@/api/ai/chat";
import {getToken} from "@/utils/auth";
import AiMessage from "./components/AiMessage.vue";
import ProductMessage from './components/ProductMessage.vue';
import ChatImage from './components/ChatImage.vue';
import {conditionText} from '@/views/bid/productParameters/presentation';
import {messageCopyText} from './messageCopy';
import {applyProductEvent, restoreProductEvents} from './productEvents';
import {listLibraries} from '@/api/bid/product';
import {Picture, DocumentCopy, Delete} from "@element-plus/icons-vue";
import {v4 as uuidv4} from "uuid";
import {useResizeObserver} from "@vueuse/core";
import {getUseMonaco} from 'markstream-vue'
import {h} from 'vue';
import { useRouter } from 'vue-router';
// 在 const {proxy} = getCurrentInstance(); 附近加上：


getUseMonaco()

const {proxy} = getCurrentInstance();
const router = useRouter();

// 退出当前账号并返回登录页，避免仅清理状态但仍停留在聊天界面。
async function handleLogout() {
  try {
    await ElMessageBox.confirm('确定要退出当前账号吗？', '退出登录', {
      confirmButtonText: '退出登录',
      cancelButtonText: '取消',
      type: 'warning'
    });
    await account.logOut();
    await router.replace('/login');
  } catch (error) {
    // 用户取消确认时不提示错误；退出失败时保留当前页面并提示。
    if (error !== 'cancel' && error !== 'close') {
      proxy.$modal?.msgError?.(error?.message || '退出登录失败，请重试');
    }
  }
}

// ===== 欢迎页状态 =====
const showWelcome = ref(true);
const startingChat = ref(false);
const checkingRagflowChat = ref(true);
const ragflowReady = ref(false);
const ragflowKnowledgeBases = ref([]);
const ragflowLibraryIds = ref([]);
const ragflowKnowledgeBasesLoading = ref(false);
const ragflowKnowledgeBasesSaving = ref(false);
const ragflowKnowledgeBasesError = ref('');
let ragflowKnowledgeBasesRequest = 0;
let ragflowKnowledgeBasesSavePending = false;
// 需要在知识库列表中排除的知识库类型
const PRODUCT_PARAMETER_LIBRARY_TYPE = 'PRODUCT_PARAMETER';
let ragflowEntryRequest = 0;
let evidenceFileRequest = null;
let evidenceDragTimer = null;
let evidenceDrag = null;
let suppressEvidenceOpen = false;

// ===== 搜索弹窗 =====
const searchDialogVisible = ref(false);
const searchInputRef = ref(null);

const isDeepThink = ref(false)
function handleDeepThinkChange(value) {
  isDeepThink.value = value
  // TODO: 这里可以预留后续对接后端的逻辑
  // 比如把 isDeepThink.value 放进请求参数里
}

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
const unavailableKnowledgeBaseCount = computed(() => {
  return ragflowKnowledgeBases.value.filter(kb => kb.hasParsedFiles === false).length;
});

// 自定义收起图标（左窄右宽）
const SidebarCollapseIcon = (props) => h('svg', {
  viewBox: '0 0 24 24',
  width: props.size || 18,
  height: props.size || 18,
  fill: 'none',
  stroke: 'currentColor',
  'stroke-width': 2,
  'stroke-linecap': 'round',
  'stroke-linejoin': 'round'
}, [
  h('rect', {x: 3, y: 3, width: 18, height: 18, rx: 4}),
  h('line', {x1: 9, y1: 3, x2: 9, y2: 21})
]);

// 自定义展开图标（左宽右窄，用于收起后的展开操作）
const SidebarExpandIcon = (props) => h('svg', {
  viewBox: '0 0 24 24',
  width: props.size || 18,
  height: props.size || 18,
  fill: 'none',
  stroke: 'currentColor',
  'stroke-width': 2,
  'stroke-linecap': 'round',
  'stroke-linejoin': 'round'
}, [
  h('rect', {x: 3, y: 3, width: 18, height: 18, rx: 4}),
  h('line', {x1: 15, y1: 3, x2: 15, y2: 21})
]);

function openKnowledgeBaseDialog() {
  if (ragflowKnowledgeBasesLoading.value) return;
  const selectableIds = new Set(
      ragflowKnowledgeBases.value
          .filter(kb => kb.hasParsedFiles !== false)
          .map(kb => kb.libraryId)
  );
  tempSelectedKnowledgeBaseIds.value = ragflowLibraryIds.value.filter(id => selectableIds.has(id));
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

async function enterExistingRagflowChat() {
  if (chatMode.value !== 'chat') return;
  if (!account.id || !account.token) {
    checkingRagflowChat.value = false;
    return;
  }
  const owner = accountGeneration, request = ++ragflowEntryRequest;
  const current = () => owner === accountGeneration && request === ragflowEntryRequest && chatMode.value === 'chat';
  checkingRagflowChat.value = true;
  try {
    const {data} = await getRagflowChatStatus();
    if (!current() || !data?.exists) return;
    ragflowReady.value = true;
    await loadRagflowKnowledgeBases();
    if (!current()) return;
    await getSessions();
    if (!current()) return;
    const sessionId = await ensureRagflowSession();
    if (!current()) return;
    showWelcome.value = false;
    await loadSession(sessionId, true);
  } catch {
    // Keep the start button available when the existing-chat check cannot complete.
  } finally {
    if (current()) checkingRagflowChat.value = false;
  }
}

// =====================

const account = useUserStore();
let accountGeneration = 0;
const modelOptions = ref([]), modelLoading = ref(false), modelsLoaded = ref(false), modelError = ref('');
let modelRequest = 0;
const currentModelId = ref(undefined);
const chatMode = ref('chat'), BidKnowledgeLibraryIds = ref([]), productLibraries = ref([]), selectedProductId = ref(null);
const productLibrariesLoading = ref(false), productLibrariesError = ref('');
let BidKnowledgeLibrariesRequest;
let chatGeneration = 0;

function invalidateStream() {
  activeSourceMessage.value = null;
  BidKnowledgeLibrariesRequest?.abort();
  productLibrariesLoading.value = false;
  const message = messageList.value[messageList.value.length - 1];
  if (message?.mode === 'product' && message.productStatus === 'running') message.productStatus = 'interrupted';
  chatGeneration++;
  abortController.value?.abort();
  abortController.value = null;
  loading.value = false;
  currentRunId.value = null;
}

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
  const options = (data?.options || []).filter(kb => kb.libraryType !== PRODUCT_PARAMETER_LIBRARY_TYPE);
  const selectableIds = new Set(
      options.filter(kb => kb.hasParsedFiles !== false).map(kb => kb.libraryId)
  );
  ragflowKnowledgeBases.value = options;
  ragflowLibraryIds.value = (data?.selectedLibraryIds || []).filter(id => selectableIds.has(id));
}

async function loadRagflowKnowledgeBases() {
  if (chatMode.value !== 'chat' || !ragflowReady.value) return;
  const owner = accountGeneration, request = ++ragflowKnowledgeBasesRequest;
  const current = () => owner === accountGeneration && request === ragflowKnowledgeBasesRequest;
  ragflowKnowledgeBasesLoading.value = true;
  ragflowKnowledgeBasesError.value = '';
  try {
    const {data} = await getRagflowKnowledgeBases();
    if (current()) {
      const latestSelection = [...ragflowLibraryIds.value];
      applyRagflowKnowledgeBases(data);
      if (ragflowKnowledgeBasesSavePending) ragflowLibraryIds.value = latestSelection;
    }
  } catch (error) {
    if (current()) {
      ragflowKnowledgeBases.value = [];
      ragflowLibraryIds.value = [];
      ragflowKnowledgeBasesError.value = error?.response?.data?.detail || '知识库读取失败，请刷新后重试';
    }
  } finally {
    if (current()) {
      ragflowKnowledgeBasesLoading.value = false;
      if (ragflowKnowledgeBasesSavePending) {
        ragflowKnowledgeBasesSavePending = false;
        saveRagflowKnowledgeBases();
      }
    }
  }
}

async function saveRagflowKnowledgeBases() {
  if (ragflowKnowledgeBasesSaving.value || ragflowKnowledgeBasesLoading.value) {
    ragflowKnowledgeBasesSavePending = true;
    return;
  }
  const owner = accountGeneration, request = ++ragflowKnowledgeBasesRequest;
  ragflowKnowledgeBasesSaving.value = true;
  ragflowKnowledgeBasesError.value = '';
  try {
    const selected = [...ragflowLibraryIds.value];
    const {data} = await updateRagflowKnowledgeBases(selected);
    if (owner === accountGeneration && request === ragflowKnowledgeBasesRequest) {
      const latestSelection = [...ragflowLibraryIds.value];
      ragflowKnowledgeBases.value = (data?.options || []).filter(kb => kb.libraryType !== PRODUCT_PARAMETER_LIBRARY_TYPE);
      ragflowLibraryIds.value = latestSelection;
      proxy.$modal.msgSuccess('知识库范围已更新');
    }
  } catch (error) {
    if (owner === accountGeneration && request === ragflowKnowledgeBasesRequest) {
      const detail = error?.response?.data?.detail || '知识库更新失败，请稍后重试';
      ragflowKnowledgeBasesError.value = detail;
      proxy.$modal.msgError(detail);
      await loadRagflowKnowledgeBases();
    }
  } finally {
    if (owner === accountGeneration) {
      ragflowKnowledgeBasesSaving.value = false;
      if (ragflowKnowledgeBasesSavePending) {
        ragflowKnowledgeBasesSavePending = false;
        saveRagflowKnowledgeBases();
      }
    }
  }
}

async function loadProductLibraries() {
  if (chatMode.value !== 'product') return;
  BidKnowledgeLibrariesRequest?.abort();
  const request = BidKnowledgeLibrariesRequest = new AbortController(), epoch = chatGeneration;
  const selected = [...BidKnowledgeLibraryIds.value];
  productLibraries.value = [];
  BidKnowledgeLibraryIds.value = [];
  productLibrariesLoading.value = true;
  productLibrariesError.value = '';
  try {
    const rows = [];
    let page = 1;
    while (true) {
      const {data} = await listLibraries({page, pageSize: 100}, request.signal);
      if (request.signal.aborted || epoch !== chatGeneration) return;
      rows.push(...data.rows);
      if (page * 100 >= data.total) break;
      page++;
    }
    productLibraries.value = rows;
    const allowed = new Set(rows.filter(row => row.status === 'ACTIVE').map(row => row.libraryId));
    BidKnowledgeLibraryIds.value = selected.filter(id => allowed.has(id));
    if (BidKnowledgeLibraryIds.value.length !== selected.length) selectedProductId.value = null;
    if (!allowed.size) productLibrariesError.value = '暂无可用于查询的资料库，请确认权限和资料库状态。';
  } catch {
    if (!request.signal.aborted && epoch === chatGeneration) {
      selectedProductId.value = null;
      productLibrariesError.value = '资料库读取失败，请检查网络和权限后刷新。';
    }
  } finally {
    if (!request.signal.aborted && epoch === chatGeneration) productLibrariesLoading.value = false;
  }
}

async function chooseProduct(id, message) {
  if (loading.value || productLibrariesLoading.value || sessionDetailError.value) return;
  if (message.sessionId !== currentSessionId.value || !message.requestMessage || !message.libraryIds?.length
      || message.parameterResult?.productStatus !== 'NEEDS_CONFIRMATION'
      || !message.parameterResult.choices?.some(choice => choice.productId === id)) return;
  if (inputMessage.value.trim() || inputImages.value.length) {
    proxy.$modal.msgError('请先处理当前输入草稿，再确认历史问题的产品');
    return;
  }
  if (!currentModelId.value) {
    proxy.$modal.msgError('请先选择模型');
    return;
  }
  chatMode.value = 'product';
  BidKnowledgeIds.value = [...message.libraryIds];
  selectedProductId.value = id;
  inputMessage.value = message.requestMessage;
  inputImages.value = [...(message.requestImages || [])];
  restoreQueryConditions(message.requestConditions);
  await handleSend();
}

async function retryProduct(message, supplementOnly = false) {
  if (loading.value || productLibrariesLoading.value || sessionDetailError.value) return;
  if (!(supplementOnly ? message.productStatus === 'completed' && message.supplementFailed : ['failed', 'stopped'].includes(message.productStatus)) || !message.generationId
      || message.sessionId !== currentSessionId.value || !message.requestMessage || !message.libraryIds?.length) return;
  if (inputMessage.value.trim() || inputImages.value.length) {
    proxy.$modal.msgError('请先处理当前输入草稿，再重新查询历史问题');
    return;
  }
  if (!currentModelId.value) {
    proxy.$modal.msgError('请先选择模型');
    return;
  }
  chatMode.value = 'product';
  BidKnowledgeLibraryIds.value = [...message.libraryIds];
  selectedProductId.value = message.parameterResult?.productStatus === 'CONFIRMED' ? message.parameterResult.productId : null;
  inputMessage.value = message.requestMessage;
  inputImages.value = [...(message.requestImages || [])];
  restoreQueryConditions(message.requestConditions);
  await handleSend(undefined, supplementOnly ? message.generationId : null);
}

function hideProductMessage(message) {
  Object.assign(message, {
    content: '',
    reasoningContent: '',
    reasoningStatus: 'completed',
    references: [],
    parameterResult: null,
    currentParameterStatuses: {},
    productStatus: 'failed',
    productErrors: ['此轮资料当前不可访问，请确认权限。']
  });
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
  } finally {
    if (epoch === chatGeneration) loading.value = false;
  }
}

async function recoverProduct(message) {
  const epoch = chatGeneration;
  try {
    const {data} = await getProductGeneration(message.sessionId, message.generationId);
    if (epoch === chatGeneration) {
      if (data.sessionId !== message.sessionId || data.generationId !== message.generationId) throw new Error('历史身份不匹配');
      Object.assign(message, restoreProductEvents(data));
    }
  } catch (error) {
    if (epoch !== chatGeneration) return;
    if ([401, 403, 404].includes(error?.response?.status)) {
      hideProductMessage(message);
    }
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
  queryConditionRows.value = fields.filter(([, item]) => typeof item === 'string' || typeof item === 'boolean').map(([key, item]) => ({
    key,
    value: item,
    type: typeof item === 'boolean' ? 'boolean' : 'text'
  }));
  queryConditionMode.value = !value ? 'any' : value.unknown || queryConditionRows.value.length !== fields.length ? 'history' : fields.length ? 'specified' : 'none';
}

function queryConditionsPayload() {
  if (queryConditionMode.value === 'any') return undefined;
  if (queryConditionMode.value === 'none') return {unknown: false};
  if (queryConditionMode.value === 'history') {
    const validate = value => {
      if (typeof value === 'number' && !Number.isSafeInteger(value)) throw new Error('历史条件中的数值无法安全重用，请重新填写精确文本后查询。');
      if (value && typeof value === 'object') Object.values(value).forEach(validate);
    };
    validate(queryHistoryConditions.value);
    return JSON.parse(JSON.stringify(queryHistoryConditions.value));
  }
  if (!queryConditionRows.value.length) throw new Error('请填写具体条件，或选择未限定条件。');
  const result = {unknown: false};
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

// 将可引用的知识库排在前面，不可引用的排在后面（保持各自原有顺序）
const sortedRagflowKnowledgeBases = computed(() => {
  return [...ragflowKnowledgeBases.value].sort((a, b) => {
    const aUnavailable = a.hasParsedFiles === false ? 1 : 0;
    const bUnavailable = b.hasParsedFiles === false ? 1 : 0;
    return aUnavailable - bUnavailable;
  });
});

function setSourcePanel(message, open) {
  if (open) activeSourceMessage.value = message;
  else if (activeSourceMessage.value === message) activeSourceMessage.value = null;
}

function toggleSessionSidebar() {
  if (activeSourceMessage.value) {
    activeSourceMessage.value = null;
    sidebarCollapsed.value = false;
  } else sidebarCollapsed.value = !sidebarCollapsed.value;
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

const defaultUserConfig = {...userConfig};

const currentModelInfo = computed(() => {
  if (!currentModelId.value) return null;
  return modelOptions.value.find((m) => m.modelId === currentModelId.value);
});

const configLoading = ref(false), configLoaded = ref(false), configSaving = ref(false), configError = ref('');
let configRequest = 0;

async function loadUserConfig() {
  if (configSaving.value) return;
  const owner = accountGeneration, request = ++configRequest;
  configLoading.value = true;
  configLoaded.value = false;
  configError.value = '';
  try {
    const {data} = await getUserChatConfig();
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
  const payload = {...editingUserConfig};
  configSaving.value = true;
  configError.value = '';
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
  modelLoading.value = true;
  modelsLoaded.value = false;
  modelError.value = '';
  modelOptions.value = [];
  currentModelId.value = undefined;
  try {
    const res = await listModelAll();
    if (!current()) return;
    modelOptions.value = res.data || [];
    modelsLoaded.value = true;
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
  const {data} = await createRagflowChatSession(defaultRagflowSessionName());
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
  loading.value = true;
  sessionError.value = '';
  try {
    const {data} = await createRagflowChatSession(defaultRagflowSessionName());
    if (owner !== accountGeneration) return;
    sessionList.value = [data, ...sessionList.value];
    currentSessionId.value = data.sessionId;
    messageList.value = data.messages || [];
    inputMessage.value = '';
    inputImages.value = [];
    sessionDetailError.value = '';
    currentSessionAgentData.value = null;
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
  sessionLoading.value = true;
  sessionError.value = '';
  sessionList.value = [];
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
  invalidateStream();
  const epoch = chatGeneration;
  currentSessionId.value = sessionId;
  messageList.value = [];
  loading.value = true;
  selectedProductId.value = null;
  currentSessionAgentData.value = null;
  sessionDetailError.value = '';
  BidKnowledgeLibraryIds.value = [];
  try {
    const request = chatMode.value === 'chat'
        ? getRagflowChatSession(sessionId)
        : getChatSession(sessionId);
    const {data} = await request;
    if (epoch !== chatGeneration) return;
    messageList.value = data.messages || [];
    if (chatMode.value === 'chat') {
      currentSessionAgentData.value = null;
      isAutoScroll.value = true;
      scrollToBottom();
      return;
    }
    currentSessionAgentData.value = data.agentData;
    chatMode.value = data.productGenerationIds?.length ? 'product' : 'chat';
    for (const generationId of data.productGenerationIds || []) {
      try {
        const {data: record} = await getProductGeneration(sessionId, generationId);
        if (epoch !== chatGeneration) return;
        const message = restoreProductEvents(record);
        messageList.value.push({
          role: 'user',
          content: message.requestMessage || '产品参数查询',
          images: message.requestImages || []
        }, message);
        BidKnowledgeLibraryIds.value = [...record.libraryIds];
        selectedProductId.value = message.parameterResult?.productStatus === 'CONFIRMED' ? message.parameterResult.productId : null;
      } catch {
        if (epoch !== chatGeneration) return;
        messageList.value.push({
          role: 'assistant',
          mode: 'product',
          sessionId,
          generationId,
          content: '',
          productStatus: 'interrupted',
          productErrors: ['此轮结果当前不可读取，请确认资料权限或稍后重试。']
        });
      }
    }
    if (epoch !== chatGeneration) return;
    loadProductLibraries();
    isAutoScroll.value = true;
    scrollToBottom();
  } catch {
    if (epoch === chatGeneration) sessionDetailError.value = '会话读取失败，请检查网络和权限后重试，或新建对话。';
  } finally {
    if (epoch === chatGeneration) loading.value = false;
  }
}

async function handleRenameSession(session) {
  const owner = accountGeneration;
  try {
    const {value} = await ElMessageBox.prompt('请输入会话名称', '重命名会话', {
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
  } finally {
    if (owner === accountGeneration) renamingSessionId.value = null;
  }
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
      .catch(() => {
      });
}

function handleDeleteMessage(message) {
  if (chatMode.value !== 'chat' || !message?.id || loading.value) return;
  const owner = accountGeneration, sessionId = currentSessionId.value;
  proxy.$modal.confirm('是否确认删除这条消息？')
      .then(() => {
        if (owner !== accountGeneration || sessionId !== currentSessionId.value) throw new Error('当前会话已变化');
        return deleteRagflowChatMessage(sessionId, message.id);
      })
      .then(({data}) => {
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
  if (isProduct) Object.assign(targetMessage, {
    mode: 'product',
    productStatus: 'running',
    sessionId: currentSessionId.value
  });
  if (isProduct) {
    currentSessionId.value ||= uuidv4();
    Object.assign(targetMessage, {
      sessionId: currentSessionId.value,
      generationId: uuidv4(),
      messageId: uuidv4(),
      retrievalOnly: true,
      requestMessage: text,
      requestImages: imageList,
      requestConditions: conditions ?? null,
      libraryIds: [...BidKnowledgeLibraryIds.value]
    });
  }
  const requestBody = isProduct ? {
    retrievalOnly: !retrySupplementFrom,
    generationId: targetMessage.generationId,
    messageId: targetMessage.messageId, ...(retrySupplementFrom ? {retrySupplementFrom} : {}),
    mode: 'product', ...(conditions === undefined ? {} : {conditions}),
    images: imageList,
    isReasoning: chatConfig.isReasoning,
    modelId: currentModelId.value,
    message: text,
    sessionId: currentSessionId.value,
    libraryIds: [...BidKnowledgeLibraryIds.value],
    productId: selectedProductId.value
  } : {
    message: text, sessionId: currentSessionId.value, isReasoning: isDeepThink.value,
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
        let body;
        try {
          body = JSON.parse(errorText);
        } catch {
          body = {};
        }
        throw Object.assign(new Error(body.userMessage || body.msg || '产品查询请求失败，请检查权限及输入'), {status: response.status});
      }
      let detail = errorText;
      try {
        detail = JSON.parse(errorText).detail || errorText;
      } catch { /* Keep the raw response. */
      }
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
      const {done, value} = await reader.read();
      if (epoch !== chatGeneration || controller.signal.aborted) return;
      if (done) break;

      buffer += decoder.decode(value, {stream: true});
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
          } else if (data.type === "session_title") {
            const session = sessionList.value.find((s) => s.sessionId === data.sessionId);
            if (session) session.sessionTitle = data.sessionTitle;
            else needRefreshSessions = true;
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
        const {data} = await getRagflowChatSession(sessionId);
        if (epoch === chatGeneration && sessionId === currentSessionId.value) {
          const persisted = data.messages || [];
          let lastUserIndex = -1;
          persisted.forEach((message, index) => {
            if (message.role === 'user' && message.content === text) lastUserIndex = index;
          });
          const hasCurrentAnswer = persisted.slice(lastUserIndex + 1)
              .some(message => message.role === 'assistant' && message.content);
          // RAGFlow 可能先返回 done、后落库回答，不能用只含用户消息的旧快照覆盖流式结果。
          // 落库后只回填服务端生成的 id、时间、证据和最终正文，保留本地思考过程、指标等字段，
          // 避免回答完成时深度思考、耗时/token、消息时间等区块突然消失。
          if (hasCurrentAnswer) {
            const localAnswer = messageList.value[aiMsgIndex];
            const localUser = aiMsgIndex > 0 ? messageList.value[aiMsgIndex - 1] : null;
            const persistedAnswer = persisted[lastUserIndex + 1];
            const persistedUser = persisted[lastUserIndex];
            if (localUser?.role === 'user' && persistedUser?.role === 'user') {
              localUser.id = persistedUser.id;
              localUser.createdAt = persistedUser.createdAt;
            }
            if (localAnswer?.role === 'assistant' && persistedAnswer?.role === 'assistant') {
              localAnswer.id = persistedAnswer.id;
              localAnswer.createdAt = persistedAnswer.createdAt;
              localAnswer.evidenceImages = persistedAnswer.evidenceImages || [];
              // 落库正文是最终版本（含引用编号规范化），同步过来；空内容时保留流式结果。
              if (persistedAnswer.content) localAnswer.content = persistedAnswer.content;
            }
          }
        }
      } catch { /* Keep the streamed text until the next explicit refresh. */
      }
      const localAnswer = messageList.value[aiMsgIndex];
      if (!aiContent.trim() && localAnswer && !(localAnswer.content || '').trim()) {
        localAnswer.content = 'AI 未返回内容，请重试。';
      }
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
    if (epoch === chatGeneration && abortController.value === controller) {
      loading.value = false;
      abortController.value = null;
    }
  }
}

function clearChat() {
  restoreQueryConditions(null);
  sessionDetailError.value = '';
  invalidateStream();
  selectedProductId.value = null;
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

function startEvidenceDrag(event) {
  if (event.button !== 0) return;
  clearTimeout(evidenceDragTimer);
  const container = event.currentTarget;
  const pointerId = event.pointerId;
  evidenceDragTimer = setTimeout(() => {
    evidenceDrag = {
      container,
      pointerId,
      startX: event.clientX,
      scrollLeft: container.scrollLeft,
      moved: false,
    };
    container.classList.add('is-dragging');
    container.setPointerCapture?.(pointerId);
  }, 260);
}

function moveEvidenceDrag(event) {
  if (!evidenceDrag || evidenceDrag.pointerId !== event.pointerId) return;
  const distance = event.clientX - evidenceDrag.startX;
  if (Math.abs(distance) > 3) evidenceDrag.moved = true;
  evidenceDrag.container.scrollLeft = evidenceDrag.scrollLeft - distance;
}

function endEvidenceDrag(event) {
  clearTimeout(evidenceDragTimer);
  evidenceDragTimer = null;
  if (!evidenceDrag || evidenceDrag.pointerId !== event.pointerId) return;
  const {container, pointerId, moved} = evidenceDrag;
  container.classList.remove('is-dragging');
  container.releasePointerCapture?.(pointerId);
  evidenceDrag = null;
  if (moved) {
    suppressEvidenceOpen = true;
    setTimeout(() => {
      suppressEvidenceOpen = false;
    }, 0);
  }
}

function suppressEvidenceClick(event) {
  if (!suppressEvidenceOpen) return;
  event.preventDefault();
  event.stopPropagation();
}

async function openEvidenceFile(evidence, message) {
  if (suppressEvidenceOpen) return;
  if (!evidence?.fileId || !evidence?.documentId || !message?.id) {
    proxy.$modal.msgWarning('未找到该证据所属文件');
    return;
  }
  const owner = accountGeneration, sessionId = currentSessionId.value;
  const fileWindow = window.open('', '_blank');
  evidenceFileRequest?.abort();
  const controller = evidenceFileRequest = new AbortController();
  try {
    const blob = await downloadRagflowEvidenceFile(
        sessionId, message.id, evidence.documentId, controller.signal
    );
    if (controller.signal.aborted || owner !== accountGeneration || sessionId !== currentSessionId.value) {
      fileWindow?.close();
      return;
    }
    const url = URL.createObjectURL(blob);
    if (fileWindow) fileWindow.location.href = url;
    else window.open(url, '_blank');
    setTimeout(() => URL.revokeObjectURL(url), 60_000);
  } catch (error) {
    fileWindow?.close();
    if (!controller.signal.aborted) {
      proxy.$modal.msgError(error?.response?.data?.detail || error?.message || '打开文件失败');
    }
  } finally {
    if (evidenceFileRequest === controller) evidenceFileRequest = null;
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
}, {flush: 'sync'});

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
            headers: {Authorization: "Bearer " + getToken()},
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
  if (chatMode.value === 'product' && productLibrariesLoading.value) {
    proxy.$modal.msgError('正在核对资料库范围，请稍候');
    return;
  }
  if (sessionDetailError.value) {
    proxy.$modal.msgError('请先重新读取当前会话，或新建对话');
    return;
  }
  if (loading.value || event?.isComposing) return;
  const text = inputMessage.value.trim();
  const images = chatMode.value === 'product' ? inputImages.value : [];
  if (chatMode.value === 'product' && images.length && userConfig.visionEnabled !== '0') {
    proxy.$modal.msgError('视觉功能已关闭，请移除图片后发送');
    return;
  }
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
    try {
      await ensureRagflowSession();
    } catch (error) {
      proxy.$modal.msgError(error?.response?.data?.detail || '新建对话失败，请稍后重试');
      return;
    }
  }

  if (chatMode.value === 'product' && (!BidKnowledgeLibraryIds.value.length || !text)) {
    proxy.$modal.msgError('产品模式需要选择资料库并输入本轮要查询或核对的问题');
    return;
  }
  let conditions;
  try {
    if (chatMode.value === 'product') conditions = queryConditionsPayload();
  } catch (error) {
    proxy.$modal.msgError(error.message);
    return;
  }
  const imageList = images.slice();
  messageList.value.push({role: "user", content: text, images: imageList});
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
          .then(() => {
            if (message?.mode === 'product' && messageList.value.includes(message)) return recoverProduct(message);
          })
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
  const {scrollTop, scrollHeight, clientHeight} = e.target;
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
  accountGeneration++;
  clearChat();
  ElMessageBox.close();

  // 关键改动：退出时不要显示欢迎页，保持空白交由路由处理
  showWelcome.value = false;

  startingChat.value = false;
  checkingRagflowChat.value = false;
  ragflowReady.value = false;
  ragflowEntryRequest++;
  evidenceFileRequest?.abort();
  evidenceFileRequest = null;
  clearTimeout(evidenceDragTimer);
  evidenceDragTimer = null;
  evidenceDrag = null;
  ragflowKnowledgeBases.value = [];
  ragflowLibraryIds.value = [];
  ragflowKnowledgeBasesLoading.value = false;
  ragflowKnowledgeBasesSaving.value = false;
  ragflowKnowledgeBasesError.value = '';
  ragflowKnowledgeBasesRequest++;
  ragflowKnowledgeBasesSavePending = false;
  sessionSearch.value = '';
  renamingSessionId.value = null;
  sidebarCollapsed.value = false;
  inputMessage.value = '';
  inputImages.value = [];
  sessionList.value = [];
  sessionLoading.value = false;
  sessionError.value = '';
  sessionRequest++;
  BidKnowledgeLibraryIds.value = [];
  productLibraries.value = [];
  productLibrariesError.value = '';
  chatMode.value = 'chat';
  modelOptions.value = [];
  currentModelId.value = undefined;
  showConfigDialog.value = false;
  modelLoading.value = false;
  modelsLoaded.value = false;
  modelError.value = '';
  modelRequest++;
  configRequest++;
  configLoading.value = false;
  configLoaded.value = false;
  configSaving.value = false;
  configError.value = '';
  Object.assign(userConfig, defaultUserConfig);
  Object.assign(editingUserConfig, defaultUserConfig);
  chatConfig.temperature = undefined;
  chatConfig.isReasoning = true;
  if (imageInputRef.value) imageInputRef.value.value = '';
  proxy.$modal.closeLoading();
}

watch(() => [account.id, account.token], ([newId, newToken], [oldId, oldToken]) => {
  // 如果当前没有 token（无论是因为退出登录，还是过期）
  if (!newToken) {
    // 1. 清空所有状态，同时会将 showWelcome 设为 false，不显示欢迎页
    clearAccountState();

    // 2. 强制跳转到登录页
    router.push('/login').catch(() => {});
    return;
  }

  // 登录 / 切换账号
  clearAccountState();
  if (account.id && account.token) {
    // 只有有 token 才显示欢迎页
    showWelcome.value = true;

    getModels();
    getSessions();
    loadUserConfig();
    enterExistingRagflowChat();
  }
}, {flush: 'sync'});
onMounted(() => {
  // 如果初始化时就没有 token，直接拦截
  if (!account.token) {
    router.push('/login').catch(() => {});
    return;
  }

  // 有 token 才正常加载
  showWelcome.value = true;
  getModels();
  getSessions();
  loadUserConfig();
  enterExistingRagflowChat();
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

.product-query-conditions {
  margin: 0 20px;
}

.query-condition-row {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin: 8px 0;
}

.query-condition-row .el-input, .query-condition-row .el-select {
  width: 200px;
}

.selected-image-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

@media (min-width: 1200px) {
  .chat-container.has-evidence-panel .chat-main {
    margin-right: min(48vw, 840px);
  }
}

.session-load-error {
  padding: 12px;
  overflow-wrap: anywhere;
}

.product-scope {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 12px;
  padding: 12px;
}

.chat-container {
  height: calc(100vh - 84px);
  padding: 0;
  background-color: var(--el-bg-color-page);
  overflow: hidden;
}

/* ===== 侧边栏美化 ===== */
.session-sidebar {
  padding: 0;
  margin: 0;
  width: 300px !important;
  min-width: 300px;
  flex: 0 0 300px;
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

        &:hover {
          color: var(--el-color-primary);
        }
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

      &:active {
        transform: translateY(0);
      }

      .btn-icon {
        margin-right: 7px;
        font-size: 17px;
      }
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

    :deep(.el-input__inner) {
      font-size: 14px;
    }
  }

  .session-list {
    flex: 1;
    min-width: 0;
    width: 100%;
    box-sizing: border-box;
    overflow-y: auto;
    padding: 4px 10px;

    &::-webkit-scrollbar {
      width: 4px;
    }

    &::-webkit-scrollbar-thumb {
      background: var(--el-border-color);
      border-radius: 2px;
    }

    .session-group-label {
      font-size: 12px;
      color: var(--el-text-color-secondary);
      padding: 0px 8px 4px;
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
        .session-actions {
          opacity: 1;
        }
      }

      &.active {
        background-color: var(--el-color-primary-light-9); /* 适配夜间模式 */
        .session-title {
          color: var(--el-color-primary);
          font-weight: 500;
        }
      }

            .session-open {
        display: flex;
        align-items: center;
        flex: 1;
        min-width: 0;
        border: 0;
        padding: 0;
        background: transparent;
        text-align: left;
        font: inherit;
        cursor: pointer;

        &:focus-visible {
          outline: 2px solid var(--el-color-primary);
          outline-offset: 2px;
        }

        /* 👇 替换这一部分：强制单行，超出隐藏并显示省略号 */
        .session-title {
          width: 100%;
          font-size: 14px;
          color: var(--el-text-color-primary);
          line-height: 1.5;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          word-break: break-all; /* 防止无空格的长串字符破坏布局 */
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

    .empty-session {
      text-align: center;
      color: var(--el-text-color-secondary);
      font-size: 13px;
      margin-top: 40px;
    }
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

    .header-title {
      font-size: 16px;
      font-weight: 600;
      color: var(--el-text-color-primary);
      margin-left: 8px;
    }

    .expand-btn {
      color: var(--el-text-color-regular);

      &:hover {
        color: var(--el-color-primary);
      }
    }

    .header-right-actions {
      display: flex;
      align-items: center;
      gap: 16px;

      .product-header-actions {
        display: flex;
        align-items: center;
      }
    }
  }

  .chat-history {
    flex: 1;
    overflow-y: auto;
    padding: 20px;

    .chat-content {
      min-height: 100%;
      padding-bottom: 20px;

      &.is-empty {
        display: flex;
        flex-direction: column;
        height: 100%;
      }
    }

    .welcome-screen {
      flex: 1;
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      color: var(--el-text-color-secondary);
      opacity: 0.8;

      .welcome-icon {
        background: var(--el-fill-color);
        border-radius: 50%;
        padding: 20px;
        margin-bottom: 20px;
        color: var(--el-color-primary);
      }

      h2 {
        margin-bottom: 10px;
        font-weight: 500;
      }
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

        .avatar-user {
          background-color: var(--el-color-primary);
        }

        .avatar-ai {
          background-color: var(--el-color-success);
        }
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

        .message-time {
          font-size: 11px;
          opacity: 0.8;
        }

        .message-bubble {
          padding: 12px 16px;
          border-radius: 12px;
          font-size: 15px;
          line-height: 1.6;
          max-width: 100%;
          min-width: 60px;
        }

        .ragflow-evidence {
          display: grid;
          grid-auto-flow: column;
          grid-auto-columns: calc((100% - 30px) / 4);
          gap: 10px;
          margin-top: 12px;
          overflow-x: auto;
          overflow-y: hidden;
          cursor: grab;
          user-select: none;
          scrollbar-width: none;
          touch-action: pan-y;

          &.is-dragging {
            cursor: grabbing;
          }

          &::-webkit-scrollbar {
            display: none;
          }
        }

        .ragflow-evidence-item {
          position: relative;
          min-width: 0;
          aspect-ratio: 4 / 3;
          margin: 0;
          border: 1px solid var(--el-border-color-light);
          border-radius: 8px;
          overflow: hidden;
          background: var(--el-fill-color-lighter);
        }

        .ragflow-evidence-image {
          display: block;
          width: 100%;
          height: 100%;
          cursor: zoom-in;
        }

        .ragflow-evidence-number {
          position: absolute;
          top: 6px;
          right: 6px;
          z-index: 1;
          min-width: 24px;
          padding: 2px 5px;
          border-radius: 999px;
          background: var(--el-color-primary);
          color: #fff;
          font-size: 12px;
          line-height: 1.3;
          text-align: center;
        }

        .ragflow-evidence-file {
          position: absolute;
          left: 6px;
          bottom: 6px;
          z-index: 1;
          max-width: calc(100% - 12px);
          padding: 3px 7px;
          overflow: hidden;
          border: 0;
          border-radius: 5px;
          background: rgba(0, 0, 0, 0.68);
          color: #fff;
          font: inherit;
          font-size: 12px;
          line-height: 1.3;
          text-align: left;
          text-overflow: ellipsis;
          white-space: nowrap;
          cursor: pointer;

          &:hover {
            background: rgba(0, 0, 0, 0.82);
          }
        }

        .message-footer {
          margin-top: 6px;
          display: flex;
          justify-content: space-between;
          align-items: center;
          width: 100%;

          .message-metrics {
            font-size: 12px;
            color: var(--el-text-color-secondary);
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
          }

          .footer-actions {
            display: flex;
            align-items: center;
            gap: 8px
          }

          :deep(.el-button) {
            padding: 0;
            margin: 0;
          }

          .model-info {
            margin-left: auto;
          }
        }
      }

      &.message-user {
        flex-direction: row-reverse;
        padding-left: 52px;

        .message-avatar {
          margin-left: 12px;
          margin-right: 0;
        }

        .message-content-wrapper {
          align-items: flex-end;

          .message-sender {
            flex-direction: row-reverse;
          }

          .message-bubble {
            background-color: var(--el-color-primary);
            color: #fff;
            border-top-right-radius: 2px;

            .user-text {
              white-space: pre-wrap;
              word-break: break-word;
            }

            .user-images {
              display: flex;
              flex-wrap: wrap;
              gap: 8px;
              margin-bottom: 8px;
              justify-content: flex-end;

              .user-image-item {
                width: 100px;
                height: 100px;
                border-radius: 4px;
                cursor: pointer;
                background-color: var(--app-surface-overlay, rgba(255, 255, 255, 0.1));
              }
            }
          }

          .message-footer {
            justify-content: flex-end;

            .footer-actions {
              flex-direction: row-reverse;
            }
          }
        }
      }

      &.message-ai {
        padding-right: 52px;

        .message-content-wrapper {
          align-items: stretch;

          .message-bubble {
            background-color: var(--el-bg-color);
            border: 1px solid var(--el-border-color);
            border-top-left-radius: 2px;
            box-shadow: 0 2px 6px rgba(0, 0, 0, 0.02);
          }
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

          .kb-name {
            margin-right: 6px;
          }

          .kb-close {
            cursor: pointer;
            font-size: 14px;
            opacity: 0.6;
            transition: opacity 0.2s;

            &:hover {
              opacity: 1;
              color: var(--el-color-danger);
            }
          }
        }
      }

      .selected-images {
        display: flex;
        flex-wrap: wrap;
        gap: 8px;
        margin-top: 4px;

        .selected-image-item {
          width: 60px;
          height: 60px;
          border-radius: 6px;
          cursor: pointer;
          border: 1px solid var(--el-border-color);
          background-color: var(--el-fill-color-light);
        }
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

          /* 与“深度思考”保持一致的胶囊按钮样式 */
          .add-kb-btn {
            display: inline-flex;
            align-items: center;
            justify-content: center;
            gap: 4px;
            height: 24px;
            padding: 0 12px;
            border-radius: 12px;
            font-size: 12px;
            color: var(--el-text-color-regular);
            border: 1px solid var(--el-border-color);
            background-color: var(--el-bg-color);
            transition: all 0.3s;
            cursor: pointer;
            user-select: none;
            font-weight: bold;

            &:hover {
              color: var(--el-color-primary);
              border-color: var(--el-color-primary-light-7);
              background-color: var(--el-bg-color);
            }

            :deep(.el-icon) {
              font-size: 14px;
            }
          }

          .toggle-chip {
            border-radius: 999px;
            margin-left: 0;
          }
        }

        .send-btn {
          padding: 10px;
          font-size: 18px;
          transition: transform 0.2s;

          &:hover:not(:disabled) {
            transform: scale(1.05);
          }
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

/* 知识库弹窗美化 (按设计图还原) */
.kb-dialog {
  :deep(.el-dialog) {
    overflow: hidden;
    border-radius: 16px;
    box-shadow: var(--el-box-shadow-dark);
    padding: 0;
    /* 夜间模式下弹窗底色 */
    background-color: var(--el-bg-color-overlay);
  }

  :deep(.el-dialog__header) {
    margin: 0;
    padding: 24px 24px 16px;
    border-bottom: none;
  }

  :deep(.el-dialog__headerbtn) {
    top: 20px;
    right: 20px;
    font-size: 20px;
  }

  .kb-dialog-header {
    display: flex;
    align-items: center;
    gap: 12px;

    .header-icon {
      font-size: 28px;
      color: var(--el-color-primary);
    }

    .header-title {
      font-size: 20px;
      font-weight: 600;
      color: var(--el-text-color-primary);
    }
  }

  :deep(.el-dialog__body) {
    padding: 0 24px 20px;
  }

  :deep(.el-dialog__footer) {
    padding: 16px 24px 24px;
    border-top: 1px solid var(--el-border-color-lighter);
  }

  .kb-dialog-content {
    min-height: 200px;
  }

  /* 顶部横幅 */
  .kb-dialog-intro {
    display: flex;
    align-items: center;
    gap: 16px;
    margin-bottom: 20px;
    padding: 20px 24px;
    /* 使用背景色变量，兼容夜间模式 */
    border: 1px solid var(--el-color-primary-light-8);
    border-radius: 12px;
    background: linear-gradient(135deg, var(--el-color-primary-light-9), var(--el-color-primary-light-8));
    position: relative;
    overflow: hidden;

    /* 夜间模式下稍微调暗一点渐变 */
    html.dark & {
      background: linear-gradient(135deg, var(--el-bg-color-overlay), var(--el-color-primary-light-9));
    }
  }

  .kb-intro-mark {
    position: relative;
    display: flex;
    align-items: center;
    justify-content: center;
    flex: 0 0 48px;
    width: 48px;
    height: 48px;
    border-radius: 12px;
    background: linear-gradient(135deg, var(--el-color-primary-light-3), var(--el-color-primary));
    color: #fff;
    font-size: 22px;
    font-weight: 700;
    box-shadow: 0 6px 16px var(--el-color-primary-light-7);

    span {
      z-index: 2;
    }

    .sparkle {
      position: absolute;
      bottom: -2px;
      right: -2px;
      width: 16px;
      height: 16px;
      color: #fff;
      opacity: 0.8;
      z-index: 1;
    }
  }

  .kb-intro-text {
    flex: 1;
    z-index: 2;

    strong {
      display: block;
      color: var(--el-text-color-primary);
      font-size: 16px;
      font-weight: 600;
      margin-bottom: 6px;
    }

    p {
      margin: 0;
      color: var(--el-text-color-secondary);
      font-size: 13px;
    }
  }

  .kb-intro-illustration {
    position: absolute;
    right: 24px;
    top: 50%;
    transform: translateY(-50%);
    width: 80px;
    height: 80px;
    opacity: 0.6; /* 稍微降低插图透明度防止夜间刺眼 */
    pointer-events: none;
    z-index: 1;

    /* 夜间模式下将 SVG 颜色调暗 */
    html.dark & {
      filter: brightness(0.6) contrast(1.2);
    }
  }

  .kb-unavailable-alert {
    margin-bottom: 16px;
    border-radius: 8px;
    /* Element 变量自带警告色适配 */
    background-color: var(--el-color-warning-light-9);
    border: 1px solid var(--el-color-warning-light-8);
    color: var(--el-color-warning);

    :deep(.el-alert__title) {
      font-size: 14px;
    }
  }

  /* 列表区域 */
  .kb-checkbox-list {
    max-height: 400px;
    overflow-y: auto;
    padding: 2px 6px 4px 2px;

    &::-webkit-scrollbar {
      width: 6px;
    }

    &::-webkit-scrollbar-thumb {
      background: var(--el-border-color);
      border-radius: 3px;
    }

    :deep(.el-checkbox-group) {
      display: flex;
      flex-direction: column;
      gap: 12px;
    }

    /* 卡片样式 */
    .kb-checkbox-item {
      display: flex;
      align-items: center;
      width: 100%;
      margin: 0;
      padding: 5px 10px;
      border: 1px solid var(--el-border-color-light);
      border-radius: 12px;
      /* 使用变量适配夜间模式底色 */
      background: var(--el-bg-color-overlay);
      transition: all 0.2s ease;
      height: auto;

      /* 隐藏原生 checkbox 样式 */
      :deep(.el-checkbox__input) {
        position: absolute;
        opacity: 0;
        width: 0;
        height: 0;
      }

      :deep(.el-checkbox__label) {
        flex: 1;
        display: flex;
        align-items: center;
        padding: 0;
        width: 100%;
        margin: 0;
      }

      // 自定义勾选框
      &::before {
        content: '';
        display: flex;
        align-items: center;
        justify-content: center;
        width: 20px;
        height: 20px;
        border: 1px solid var(--el-border-color);
        border-radius: 4px;
        margin-right: 12px;
        flex-shrink: 0;
        transition: all 0.2s;
        background-color: var(--el-bg-color);
      }

      /* 选中态 */
      &.is-checked {
        border-color: var(--el-color-primary);
        background-color: var(--el-color-primary-light-9);
        box-shadow: 0 2px 8px var(--el-color-primary-light-8);

        &::before {
          content: '✓';
          background-color: var(--el-color-primary);
          border-color: var(--el-color-primary);
          color: #fff;
          font-size: 12px;
          font-weight: bold;
        }
      }

      /* 悬浮态 */
      &:hover:not(.is-checked):not(.is-unavailable) {
        border-color: var(--el-border-color);
        background-color: var(--el-fill-color-light);
      }

      /* 禁用不可用态 */
      &.is-unavailable {
        border-style: dashed;
        background-color: var(--el-fill-color-lighter);
        border-color: var(--el-border-color-lighter);
        cursor: not-allowed;

        &::before {
          background-color: var(--el-fill-color);
          border-color: var(--el-border-color-light);
        }

        .kb-item-name {
          color: var(--el-text-color-disabled);
        }

        .kb-item-icon {
          opacity: 0.5;
        }
      }

      /* 左侧图标 */
      .kb-item-icon {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 40px;
        height: 40px;
        border-radius: 10px;
        margin-right: 16px;
        flex-shrink: 0;
        font-size: 20px;
        color: #fff;

        /* 夜间模式下略微降低亮度 */
        html.dark & { filter: brightness(0.85); }

        &.icon-color-0 { background: linear-gradient(135deg, #74b0ff, #409eff); }
        &.icon-color-1 { background: linear-gradient(135deg, #b5a1f8, #8e72f8); }
        &.icon-color-2 { background: linear-gradient(135deg, #a1c9f8, #6b9df8); }
      }

      /* 中间信息区 */
      .kb-item-info {
        display: flex;
        flex-direction: column;
        justify-content: center;
        flex: 1;
        min-width: 0;
        gap: 6px;
        height: 40px;
      }

      .kb-item-title {
        display: flex;
        align-items: center;
        gap: 10px;
        min-width: 0;
      }

      .kb-item-name {
        font-size: 16px;
        font-weight: 600;
        color: var(--el-text-color-primary);
        line-height: 1.4;
        overflow-wrap: anywhere;
      }

      .kb-status-tag {
        font-size: 12px;
        padding: 2px 8px;
        border-radius: 4px;
        font-weight: 500;

        &.tag-success {
          color: var(--el-color-success);
          background-color: var(--el-color-success-light-9);
        }

        &.tag-disabled {
          color: var(--el-text-color-disabled);
          background-color: var(--el-fill-color);
        }
      }

      /* 右侧箭头 */
      .kb-item-arrow {
        color: var(--el-text-color-placeholder);
        font-size: 16px;
        margin-left: 12px;
        flex-shrink: 0;
        display: flex;
        align-items: center;
      }
    }
  }

  /* 底部操作区 */
  .dialog-footer {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
  }

  .kb-selection-count {
    color: var(--el-text-color-regular);
    font-size: 14px;

    .count-num {
      color: var(--el-color-primary);
      font-weight: 600;
    }
  }

  .footer-btns {
    display: flex;
    gap: 12px;
  }
}

.chat-config-dialog {
  :deep(.el-dialog__body) {
    padding-top: 10px;
    padding-bottom: 10px;
  }

  :deep(.el-form-item) {
    margin-bottom: 16px;
  }

  :deep(.el-form-item__label) {
    font-size: 13px;
    color: var(--el-text-color-secondary);
  }
}

.logout-btn {
  margin-left: 12px;
  border-radius: 8px;
}

.session-search-divider {
  flex: 0 0 auto;
  width: 260px;
  height: 1px;
  margin: 5px auto 10px;
  border-radius: 2px;
  background: var(--el-border-color);
  opacity: 0.8;
}

/* ================= 新增：深度思考按钮样式 ================= */
.deep-think-tag {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 4px;
  height: 24px; /* 和 size="small" 的 el-button 高度基本一致 */
  padding: 0 12px;
  border-radius: 12px; /* 胶囊形状 */
  font-size: 12px;
  color: var(--el-text-color-regular);
  border: 1px solid var(--el-border-color);
  background-color: var(--el-bg-color);
  transition: all 0.3s;
  cursor: pointer;
  user-select: none;

  &:hover {
    color: var(--el-color-primary);
    border-color: var(--el-color-primary-light-7);
  }

  /* 选中后的样式 */
  &.el-check-tag.is-checked {
    background-color: var(--el-color-primary-light-9);
    border-color: var(--el-color-primary);
    color: var(--el-color-primary);
    font-weight: 500;
  }

  .tag-icon {
    font-size: 14px;
  }
}
/* ======================================================= */
</style>
