下面是index.vue文件：
<template>
  <div class="app-container evidence-chain-page">
    <el-page-header title="返回任务" class="page-header" @back="goBack">
      <template #content><span>证据链详情</span></template>
      <template #extra><code>Job {{ jobId || '-' }} · {{ jobStatus || '加载中' }}</code></template>
    </el-page-header>

    <el-skeleton v-if="loading" :rows="8" animated />
    <el-empty v-else-if="!workflows.length" description="当前任务还没有可展示的 Workflow 证据" />
    <section v-for="workflow in workflows" v-else :key="workflow.workflowRunId" class="workflow-card">
      <header class="workflow-header">
        <div>
          <h3>{{ workflow.checkItemName || workflow.workflowName || workflow.workflowCode }}</h3>
          <span>{{ workflow.workflowCode || '-' }} / {{ workflow.workflowRunId }}</span>
        </div>
        <div class="workflow-tags">
          <el-tag :type="workflowType(workflow.workflowStatus)">{{ workflow.workflowStatus || '-' }}</el-tag>
          <el-tag :type="resultType(workflow.resultStatus)">{{ workflow.resultStatus || '-' }}</el-tag>
          <el-tag :type="riskType(workflow.riskLevel)">风险 {{ workflow.riskLevel || '-' }}</el-tag>
<!--          <el-tag type="info">置信度 {{ workflow.confidence ?? '-' }}</el-tag>-->
        </div>
      </header>

      <div class="processing-block">
        <el-alert
          :description="workflow.workflowStatus === 'FAILED' ? workflowFailureDescription(workflow) : ''"
          :type="alertType(workflow)"
          show-icon
          :closable="false"
        >
          <template #title>
            <template v-if="workflow.workflowStatus === 'FAILED'">
              {{ alertTitle(workflow) }}
            </template>
            <template v-else-if="workflow.resultStatus !== 'PASS'">
              <span>不通过，</span>
              <span>需人工复核</span>
            </template>
            <template v-else>
              {{ alertTitle(workflow) }}
            </template>
          </template>
        </el-alert>
        <div v-if="workflow.analysisView?.evidenceGroups?.length" class="analysis-groups">
          <section v-for="(group, index) in workflow.analysisView.evidenceGroups" :key="index" class="analysis-group">
            <h4 v-if="group.heading" class="analysis-group-title">{{ group.heading }}</h4>
            <div class="analysis-group-body">
              <template v-for="(block, blockIndex) in group.blocks" :key="blockIndex">
                <ul
                  v-if="['list', 'sublist'].includes(block.type)"
                  class="analysis-list"
                  :class="{ 'analysis-sublist': block.type === 'sublist' }"
                >
                  <li v-for="(parts, itemIndex) in block.items" :key="itemIndex">
                    <template v-for="(part, partIndex) in parts" :key="partIndex">
                      <strong v-if="part.bold" :class="toneClass(part)">{{ part.text }}</strong>
                      <el-link
                        v-else-if="isSubTaskReviewPart(part, group.heading)"
                        type="warning"
                        :underline="false"
                        class="review-jump-link"
                        @click.stop="scrollToSubTaskClaim(workflow, group.heading)"
                      >需人工复核</el-link>
                      <span v-else-if="part.tone || part.highlight" :class="toneClass(part)">{{ part.text }}</span>
                      <template v-else>{{ part.text }}</template>
                    </template>
                  </li>
                </ul>
                <p v-else class="analysis-text">
                  <template v-for="(part, partIndex) in block.parts" :key="partIndex">
                    <br v-if="part.break" />
                    <strong v-if="part.bold" :class="toneClass(part)">{{ part.text }}</strong>
                    <el-link
                      v-else-if="isSubTaskReviewPart(part, group.heading)"
                      type="warning"
                      :underline="false"
                      class="review-jump-link"
                      @click.stop="scrollToSubTaskClaim(workflow, group.heading)"
                    >需人工复核</el-link>
                    <span v-else-if="part.tone || part.highlight" :class="toneClass(part)">{{ part.text }}</span>
                    <template v-else>{{ part.text }}</template>
                  </template>
                </p>
              </template>
            </div>
          </section>
        </div>
        <p v-else class="analysis-text">
          <template v-for="(part, partIndex) in workflow.analysisView?.conclusionParts || []" :key="partIndex">
            <span v-if="part.tone || part.highlight" :class="toneClass(part)">{{ part.text }}</span>
            <template v-else>{{ part.text }}</template>
          </template>
          <template v-if="!workflow.analysisView?.conclusionParts?.length">尚未形成业务结论</template>
        </p>
        <ProcessingMask v-if="isProcessing(executionState(workflow))" />
      </div>

<!--      <div class="processing-block">-->
<!--        <el-alert-->
<!--          :title="alertTitle(workflow)"-->
<!--          :description="workflow.workflowStatus === 'FAILED' ? workflowFailureDescription(workflow) : ''"-->
<!--          :type="alertType(workflow)"-->
<!--          show-icon-->
<!--          :closable="false"-->
<!--        />-->
<!--        <ProcessingMask v-if="isProcessing(executionState(workflow))" />-->
<!--      </div>-->

<!--      <section class="processing-block analysis-section">-->
<!--        <div class="section-title">证据详情</div>-->
<!--        <el-alert-->
<!--          v-if="workflow.analysis?.status === 'FAILED'"-->
<!--          title="该审查项分析生成失败，未生成原因或者建议。"-->
<!--          :description="workflow.analysis.errorMessage"-->
<!--          type="warning"-->
<!--          :closable="false"-->
<!--          show-icon-->
<!--        />-->
<!--        <article v-else-if="workflow.analysis?.markdownContent" class="analysis-paper">-->
<!--          <section class="analysis-field">-->
<!--            <h4>结论</h4>-->
<!--            <p class="analysis-text">{{ workflow.analysisView?.conclusion || '未生成' }}</p>-->
<!--          </section>-->
<!--          <section class="analysis-field">-->
<!--            <h4>证据详情</h4>-->
<!--            <template v-for="(block, index) in workflow.analysisView?.evidenceDetail || []" :key="index">-->
<!--              <ul v-if="block.type === 'list'" class="analysis-list">-->
<!--                <li v-for="(item, itemIndex) in block.items" :key="itemIndex">{{ item }}</li>-->
<!--              </ul>-->
<!--              <p v-else class="analysis-text">-->
<!--                <strong v-if="block.heading">{{ block.heading }}：</strong>{{ block.text }}-->
<!--              </p>-->
<!--            </template>-->
<!--            <p v-if="!workflow.analysisView?.evidenceDetail?.length" class="analysis-text">未生成</p>-->
<!--          </section>-->
<!--          <section class="analysis-field">-->
<!--            <h4>人工决定</h4>-->
<!--            <p class="analysis-text">{{ workflow.analysisView?.humanDecision || '未提交' }}</p>-->
<!--          </section>-->
<!--          <section class="analysis-field">-->
<!--            <h4>复核备注</h4>-->
<!--            <p class="analysis-text">{{ workflow.analysisView?.reviewComment || '无' }}</p>-->
<!--          </section>-->
<!--          <section class="analysis-field">-->
<!--            <h4>重试建议</h4>-->
<!--            <p class="analysis-text">{{ workflow.analysisView?.retryRecommendation || '-' }}</p>-->
<!--          </section>-->
<!--        </article>-->
<!--        <el-alert-->
<!--          v-else-if="analysisState(workflow).status === 'FAILED'"-->
<!--          title="报告单项分析处理失败"-->
<!--          :description="analysisState(workflow).errorMessage"-->
<!--          type="error"-->
<!--          :closable="false"-->
<!--          show-icon-->
<!--        />-->
<!--        <el-empty v-else description="尚未生成报告单项分析" :image-size="48" />-->
<!--        <ProcessingMask v-if="isProcessing(analysisState(workflow))" />-->
<!--      </section>-->

<!--      <section class="processing-block failure-diagnostics">-->
<!--        <div class="section-title">运行诊断</div>-->
<!--        <el-alert-->
<!--          v-for="diagnostic in workflow.diagnostics || []"-->
<!--          :key="diagnostic.evidenceId"-->
<!--          :title="diagnostic.locator?.text || 'Workflow 执行异常'"-->
<!--          :description="[diagnostic.meta?.errorStage, diagnostic.meta?.errorCode || diagnostic.meta?.errorType, diagnostic.sourceService].filter(Boolean).join(' / ')"-->
<!--          type="error"-->
<!--          :closable="false"-->
<!--          show-icon-->
<!--          class="diagnostic-row"-->
<!--        />-->
<!--        <el-empty v-if="!workflow.diagnostics?.length" description="暂无运行诊断" :image-size="48" />-->
<!--        <ProcessingMask v-if="isProcessing(executionState(workflow))" />-->
<!--      </section>-->

      <section
        :id="`claims-section-${workflow.workflowRunId}`"
        class="processing-block claims-section"
      >
        <div class="section-title" style="margin-top: 50px">事实主张</div>
        <p v-if="evidenceColorLegend(workflow.workflowCode)" class="evidence-legend">
          {{ evidenceColorLegend(workflow.workflowCode) }}
        </p>
        <el-empty v-if="!visibleClaims(workflow).length" description="Workflow 未产出 Claim" :image-size="56" />
        <article
          v-for="claim in visibleClaims(workflow)"
          :id="claimCardDomId(workflow.workflowRunId, claim.claimId)"
          :key="claim.claimId"
          class="claim-card"
          :class="{ 'is-claim-highlighted': highlightedClaimKey === claimHighlightKey(workflow, claim) }"
        >
          <div class="claim-header">
            <div><strong>{{ claim.statement }}</strong><code>{{ claim.claimId }}</code></div>
            <el-tag :type="claim.evidenceSufficient ? 'success' : 'warning'">
              {{ claim.evidenceSufficient ? '证据充分' : '证据不足' }}
            </el-tag>
          </div>
          <el-alert v-if="!claim.evidence?.length" title="该 Claim 没有关联证据" type="warning" :closable="false" show-icon />
          <div class="evidence-grid">
            <EvidenceCard
              v-for="evidence in claim.evidence || []"
              :key="evidence.evidenceId"
              :evidence="evidence"
              :workflow-code="workflow.workflowCode"
              @preview-artifact="previewArtifact"
            />
          </div>
        </article>
<!--        <div v-if="workflow.context?.length" class="context-section">-->
<!--          <div class="section-title">补充上下文</div>-->
<!--          <div class="evidence-grid">-->
<!--            <EvidenceCard-->
<!--              v-for="evidence in workflow.context"-->
<!--              :key="evidence.evidenceId"-->
<!--              :evidence="evidence"-->
<!--              :workflow-code="workflow.workflowCode"-->
<!--              @preview-artifact="previewArtifact"-->
<!--            />-->
<!--          </div>-->
<!--        </div>-->
        <ProcessingMask v-if="isProcessing(executionState(workflow))" />
      </section>

      <section class="processing-block tool-summary-section">
        <div class="section-title tool-summary-heading">
          <span>Tool 调用摘要（{{ workflow.toolCalls?.length || 0 }}）</span>
          <button type="button" class="tool-summary-toggle" :class="{ 'is-expanded': expandedToolSummaries.has(workflow.workflowRunId) }"
            :aria-expanded="expandedToolSummaries.has(workflow.workflowRunId)"
            :aria-controls="`tool-summary-${workflow.workflowRunId}`"
            :aria-label="expandedToolSummaries.has(workflow.workflowRunId) ? '收起 Tool 调用摘要' : '展开 Tool 调用摘要'"
            :title="expandedToolSummaries.has(workflow.workflowRunId) ? '收起 Tool 调用摘要' : '展开 Tool 调用摘要'"
            @click="toggleToolSummary(workflow.workflowRunId)">
            <el-icon><ArrowDown /></el-icon>
          </button>
        </div>
        <div :id="`tool-summary-${workflow.workflowRunId}`">
        <el-table v-if="workflow.toolCalls?.length" :data="visibleToolCalls(workflow)" border size="small">
          <el-table-column prop="sequence" label="#" width="56" />
          <el-table-column prop="title" label="调用" min-width="180" />
          <el-table-column prop="status" label="状态" width="110" />
          <el-table-column prop="durationMs" label="耗时(ms)" width="110" />
          <el-table-column prop="error" label="错误" min-width="220" show-overflow-tooltip />
          <el-table-column label="详情" width="90">
            <template #default="{ row }"><el-link type="primary" :underline="false" @click="openToolCall(workflow, row)">查看</el-link></template>
          </el-table-column>
        </el-table>
        <el-empty v-else description="暂无 Tool 调用记录" :image-size="48" />
        </div>
        <ProcessingMask v-if="isProcessing(executionState(workflow))" />
      </section>
    </section>

<!--    <section class="report-section processing-block">-->
<!--      <header class="report-header">-->
<!--        <div>-->
<!--          <h3>关联报告</h3>-->
<!--          <span v-if="report.reportId">V{{ report.versionNo }} · {{ report.generationTrigger || '-' }}</span>-->
<!--          <span v-else>等待报告版本</span>-->
<!--        </div>-->
<!--        <el-tag v-if="report.reportId" :type="report.status === 'COMPLETE' ? 'success' : 'warning'">{{ report.status === 'COMPLETE' ? '完整报告' : '部分完成' }}</el-tag>-->
<!--      </header>-->
<!--      <el-alert-->
<!--        v-if="report.status === 'PARTIAL'"-->
<!--        title="该版本为部分完成报告；报告内容不能替代上方 Workflow 失败诊断。"-->
<!--        type="warning"-->
<!--        :closable="false"-->
<!--        show-icon-->
<!--      />-->
<!--      <article class="report-paper">-->
<!--        <MarkdownRender v-if="report.content" :content="report.content" :is-dark="settingsStore.isDark && !printing" />-->
<!--        <el-alert-->
<!--          v-else-if="reportState.status === 'FAILED'"-->
<!--          title="关联报告生成失败"-->
<!--          :description="reportState.errorMessage"-->
<!--          type="error"-->
<!--          :closable="false"-->
<!--          show-icon-->
<!--        />-->
<!--        <el-empty v-else description="尚未生成关联报告" />-->
<!--      </article>-->
<!--      <ProcessingMask v-if="isProcessing(reportState)" />-->
<!--    </section>-->

    <el-dialog v-model="preview.visible" :title="preview.title" width="82%" top="5vh" @closed="clearPreview">
      <div v-loading="preview.loading" class="preview-body">
        <el-image
          v-if="preview.kind === 'image' && preview.url"
          :src="preview.url"
          :preview-src-list="[preview.url]"
          preview-teleported
          fit="contain"
          class="preview-image"
        />
        <iframe v-else-if="preview.kind === 'pdf' && preview.url" :src="preview.url" class="preview-pdf" title="证据 PDF" />
        <pre v-else-if="preview.kind === 'text'" class="preview-text">{{ preview.text }}</pre>
        <el-empty v-else-if="!preview.loading" description="该产物不支持内联预览，已按下载处理" />
      </div>
    </el-dialog>
  </div>
</template>

<script setup name="BidEvidenceChain">
import useSettingsStore from '@/store/modules/settings'
const settingsStore = useSettingsStore()
const printing = ref(false)
const beforePrint = () => { printing.value = true }
const afterPrint = () => { printing.value = false }
onMounted(() => {
  window.addEventListener('beforeprint', beforePrint)
  window.addEventListener('afterprint', afterPrint)
})
onBeforeUnmount(() => {
  window.removeEventListener('beforeprint', beforePrint)
  window.removeEventListener('afterprint', afterPrint)
})
import { defineComponent, h, onBeforeUnmount, onMounted, reactive, ref } from 'vue'
import { ElButton, ElTag } from 'element-plus'
import { ArrowDown } from '@element-plus/icons-vue'
import { MarkdownRender } from 'markstream-vue'
import 'markstream-vue/index.css'
import { useRoute, useRouter } from 'vue-router'
import { getJobArtifactContent, listJobEvidenceChain } from '@/api/bid'
import {
  certificateValidityState,
  performanceEvidenceActionLabel,
  performanceEvidenceMatch,
  sealSignatureState
} from './artifacts.js'
import { analysisView, evidenceCardState, evidenceColorLegend, splitSubTaskHeading, visibleClaims } from './analysis.js'

const ProcessingMask = defineComponent({
  name: 'ProcessingMask',
  setup() {
    return () => h('div', { class: 'processing-mask', role: 'status', 'aria-live': 'polite' }, [
      h('span', { class: 'processing-ring', 'aria-hidden': 'true' }),
      h('small', '正在努力处理中')
    ])
  }
})

const EvidenceCard = defineComponent({
  name: 'EvidenceCard',
  props: {
    evidence: { type: Object, required: true },
    workflowCode: { type: String, default: '' }
  },
  emits: ['preview-artifact'],
  setup(props, { emit }) {
    return () => {
      const row = props.evidence || {}
      const locator = row.locator || {}
      const evidenceRefs = (Array.isArray(row.artifactRefs) ? row.artifactRefs : [])
        .filter(ref => ['performance_contract_evidence', 'performance_invoice_evidence'].includes(ref?.artifactType))
        .sort((left, right) => String(left.artifactType).localeCompare(String(right.artifactType)))
      const performanceMatch = performanceEvidenceMatch(locator.text)
      const workflowCode = String(props.workflowCode || '')
      const isImage = String(row.contentKind || '').toUpperCase() === 'IMAGE'
      // 后端 meta.card_state：present = 已获取到证据（绿），missing = 未获取到证据（红）
      const cardState = evidenceCardState(row)
      const cardRefs = Array.isArray(row.artifactRefs) ? row.artifactRefs : []
      // 4.3 的 IMAGE 卡片是授权委托书图片：“查看原始图片 + 印章/手写签字截图”和状态上色
      const isSealSignatureCard = workflowCode === 'seal_signature_check' && isImage
      const isCertificateCard = workflowCode === 'certificate_validity_check' && isImage
      // 4.1/3.1 的 IMAGE 卡片只补“查看原始图片”按钮（4.1 不要印章/签字截图）；
      // 4.1 未定位到章节或章节下无图片（card_state = missing）时不再提供该入口
      const isOriginalImageCard = ['auth_letter_ledger_compare', 'gsxt_screenshot_check'].includes(workflowCode)
        && isImage
        && !(workflowCode === 'auth_letter_ledger_compare' && cardState === 'missing')
      // 4.4 证书 IMAGE 卡片：按证书有效期与开标日期的先后关系上色；
      // 4.4 首页 IMAGE 卡片与 TEXT 卡片改由后端 meta.card_state 上色（取到开标日期/证据为绿，否则红）
      const validityState = isCertificateCard ? certificateValidityState(locator.text) : ''
      // 4.3 的 IMAGE 卡片：按印章与签字/说明的状态上色
      const sealState = isSealSignatureCard ? sealSignatureState(locator.text) : ''
      // 3.1/4.1/4.3/4.4 的 IMAGE 卡片：复用 4.5 的 artifact 预览，在卡片底部补“查看原始图片”按钮
      // （4.3 的裁剪产物会把 evidence.artifact_id 改成裁剪图，原始图取 SOURCE 产物，兜底 rawArtifact/artifact）
      const sourceRef = cardRefs.find(ref => ref?.artifactId && String(ref?.variant || '').toUpperCase() === 'SOURCE')
      const imageRef = (isCertificateCard || isSealSignatureCard || isOriginalImageCard)
        ? (sourceRef || (row.rawArtifact?.artifactId ? row.rawArtifact : null) || (row.artifact?.artifactId ? row.artifact : null))
        : null
      // 4.3：印章/手写签字裁剪截图（识别到才会生成，对应“如有”）
      const cropActions = isSealSignatureCard
        ? cardRefs
          .filter(ref => /_crop$/.test(String(ref?.artifactType || '')))
          .map(ref => ({
            ref,
            label: String(ref.artifactType).includes('signature') ? '查看手写签字' : '查看印章'
          }))
        : []
      const actions = [
        ...evidenceRefs.map(ref => ({ ref, label: performanceEvidenceActionLabel(ref.artifactType, performanceMatch) })),
        ...(imageRef ? [{ ref: imageRef, label: '查看原始图片' }] : []),
        ...cropActions
      ]
      return h('article', {
        class: [
          'evidence-card',
          performanceMatch && (performanceMatch.matched ? 'is-performance-match' : 'is-performance-mismatch'),
          validityState && `is-validity-${validityState}`,
          sealState && `is-seal-signature-${sealState}`,
          cardState && `is-evidence-${cardState}`
        ]
      }, [
        h('div', { class: 'evidence-card-head' }, [
          h('strong', row.evidenceKey || '证据'),
          h('div', { class: 'evidence-card-tags' }, [
            performanceMatch ? h('span', { class: 'evidence-match-status' }, performanceMatch.matched ? '✓ 匹配成功' : '✕ 匹配失败') : null,
            h(ElTag, { size: 'small', type: row.contentKind === 'ABSENCE' ? 'warning' : 'info' }, () => row.contentKind || 'TEXT'),
          ]),
        ]),
        h('dl', { class: 'locator-grid' }, [
          h('dt', '来源文件'), h('dd', locator.fileName || (locator.fileId ? `File #${locator.fileId}` : '-')),
          h('dt', '文档类型'), h('dd', locator.docType || '-'),
          ...(locator.pageNo == null ? [] : [h('dt', '页码'), h('dd', locator.pageNo)]),
          h('dt', 'bbox'), h('dd', locator.bbox ? JSON.stringify(locator.bbox) : '-'),
          h('dt', '来源服务'), h('dd', [row.sourceService, row.pageSourceService].filter(Boolean).join(' / ') || '-'),
          h('dt', '原生分'), h('dd', row.nativeConfidence ?? '-'),
        ]),
        locator.text ? h('p', { class: 'evidence-snippet' }, locator.text) : null,
        actions.length ? h('div', { class: 'artifact-actions' }, actions.map(({ ref, label }) => h(ElButton, {
          key: ref.artifactId,
          size: 'small',
          type: 'primary',
          plain: true,
          onClick: () => emit('preview-artifact', ref, { locator }),
        }, () => label))) : null,
      ])
    }
  },
})

const route = useRoute()
const router = useRouter()
const loading = ref(false)
const workflows = ref([])
const highlightedClaimKey = ref('')
let claimHighlightTimer = null
const expandedToolSummaries = reactive(new Set())
function toggleToolSummary(workflowRunId) {
  if (expandedToolSummaries.has(workflowRunId)) expandedToolSummaries.delete(workflowRunId)
  else expandedToolSummaries.add(workflowRunId)
}
function visibleToolCalls(workflow) {
  const calls = workflow.toolCalls || []
  return expandedToolSummaries.has(workflow.workflowRunId) ? calls : calls.slice(0, 3)
}
const report = ref({})
const reportState = ref({ status: 'PENDING', errorMessage: null })
const jobStatus = ref('')
// 同一路由切换 jobId/workflowRunId 时组件会被复用，必须用响应式参数
const jobId = computed(() => String(route.query.jobId || ''))
const workflowRunId = computed(() => String(route.query.workflowRunId || ''))
const preview = reactive({ visible: false, loading: false, kind: '', url: '', text: '', title: '' })

// 标题：FAILED 时显示失败摘要；否则按 resultStatus 二选一
const alertTitle = (workflow) => {
  if (workflow.workflowStatus === 'FAILED') {
    return workflow.failureSummary || '执行失败'
  }
  return workflow.resultStatus === 'PASS' ? '通过' : '不通过，需人工复核'
}

// 颜色：PASS 绿；FAILED 红；其他（不通过/处理中）黄
const alertType = (workflow) => {
  if (workflow.workflowStatus === 'FAILED') return 'error'
  return workflow.resultStatus === 'PASS' ? 'success' : 'warning'
}

// 结论关键词着色：需人工复核/不通过 = 黄，通过 = 绿
const toneClass = (part) => [
  part?.tone ? `analysis-tone-${part.tone}` : '',
  part?.highlight ? 'text-highlight' : ''
].filter(Boolean).join(' ')

function workflowFailureDescription(workflow = {}) {
  const stage = workflow.errorStage ? `阶段：${workflow.errorStage}` : ''
  const code = workflow.errorCode ? `错误码：${workflow.errorCode}` : ''
  const tool = workflow.failedTool ? `Tool：${workflow.failedTool}` : ''
  const message = workflow.errorMessage || (
    workflow.errorStage && !workflow.failedTool ? '工作流逻辑在首次 Tool 调用前失败' : '工作流执行失败'
  )
  return [stage, code, tool, message].filter(Boolean).join('；')
}
let pollTimer
let disposed = false
let loadSequence = 0

function workflowType(value) { return value === 'SUCCESS' ? 'success' : value === 'FAILED' ? 'danger' : 'info' }
function resultType(value) { return value === 'PASS' ? 'success' : value === 'FAIL' ? 'danger' : 'warning' }
function riskType(value) { return value === 'HIGH' ? 'danger' : value === 'MEDIUM' ? 'warning' : 'success' }
function isProcessing(state) { return ['PENDING', 'RUNNING'].includes(String(state?.status || '').toUpperCase()) }
function executionState(workflow) {
  return workflow.processingState?.execution || {
    status: ['SUCCESS', 'FAILED'].includes(workflow.workflowStatus) ? workflow.workflowStatus : 'PENDING',
    errorMessage: workflow.errorMessage
  }
}
function analysisState(workflow) {
  return workflow.processingState?.analysis || {
    status: workflow.analysis?.status || (isProcessing(executionState(workflow)) ? 'PENDING' : 'MISSING'),
    errorMessage: workflow.analysis?.errorMessage
  }
}
function goBack() { router.replace({ path: '/bid/index', query: { jobId: jobId.value } }) }

// Claim 卡片 DOM ID 与高亮键均包含 workflowRunId，避免不同 Workflow 的 Claim ID 重复时串位。
function claimCardDomId(workflowRunId, claimId) {
  return `claim-card-${encodeURIComponent(String(workflowRunId || ''))}-${encodeURIComponent(String(claimId || ''))}`
}

function claimHighlightKey(workflow, claim) {
  return `${workflow?.workflowRunId || ''}::${claim?.claimId || ''}`
}

function normalizeMatchText(value) {
  return String(value || '').toLowerCase().replace(/[\\s\\p{P}\\p{S}【】（）()：:、，。；;_-]+/gu, '')
}

function findClaimForSubTask(workflow, heading) {
  const subTask = splitSubTaskHeading(String(heading || ''))
  if (!subTask?.name) return null

  const taskName = normalizeMatchText(subTask.name)
  const taskNumber = String(subTask.number || '')
  const claims = visibleClaims(workflow)
  const scored = claims.map(claim => {
    const searchable = [
      claim?.claimId,
      claim?.statement,
      claim?.subTaskName,
      claim?.subTaskNumber,
      claim?.meta?.subTaskName,
      claim?.meta?.subTaskNumber
    ].map(value => String(value ?? '')).join(' ')
    const normalized = normalizeMatchText(searchable)
    const nameMatched = Boolean(taskName && normalized.includes(taskName))
    const numberMatched = Boolean(taskNumber && new RegExp(`(?:子任务|task)?0?${taskNumber}(?!\\d)`, 'i').test(searchable))
    return { claim, score: (nameMatched ? 2 : 0) + (numberMatched ? 1 : 0), nameMatched, numberMatched }
  }).filter(item => item.score > 0)

  // 名称优先；只有名称未命中时，才允许唯一编号命中，避免跳到相似子任务。
  const nameMatches = scored.filter(item => item.nameMatched)
  if (nameMatches.length === 1) return nameMatches[0].claim
  if (nameMatches.length > 1) {
    const exact = nameMatches.filter(item => normalizeMatchText(item.claim?.statement) === taskName)
    if (exact.length === 1) return exact[0].claim
    return null
  }
  const numberMatches = scored.filter(item => item.numberMatched)
  return numberMatches.length === 1 ? numberMatches[0].claim : null
}

function isSubTaskReviewPart(part, heading) {
  return String(part?.text || '').trim() === '需人工复核'
    && Boolean(splitSubTaskHeading(String(heading || ''))?.name)
}

function scrollToSubTaskClaim(workflow, heading) {
  const claim = findClaimForSubTask(workflow, heading)
  if (!claim) {
    ElMessage.warning('未能根据子任务编号和名称匹配到对应的 Claim')
    return
  }

  const target = document.getElementById(claimCardDomId(workflow.workflowRunId, claim.claimId))
  if (!target) {
    ElMessage.warning('已匹配到 Claim，但页面中未找到对应证据卡片')
    return
  }

  highlightedClaimKey.value = claimHighlightKey(workflow, claim)
  target.scrollIntoView({ behavior: 'smooth', block: 'center' })

  if (claimHighlightTimer) clearTimeout(claimHighlightTimer)
  claimHighlightTimer = setTimeout(() => {
    highlightedClaimKey.value = ''
    claimHighlightTimer = null
  }, 2200)
}

function openToolCall(workflow, call) {
  router.push({ path: '/bid/tool-calls', query: { jobId: jobId.value, workflowRunId: workflow.workflowRunId, tool: call.name, callId: call.callId, title: call.title } })
}
function mediaKind(mediaType = '', fileName = '') {
  const media = mediaType.toLowerCase()
  const name = fileName.toLowerCase()
  if (media.startsWith('image/') && media !== 'image/svg+xml') return 'image'
  if (media === 'application/pdf' || name.endsWith('.pdf')) return 'pdf'
  if (media === 'application/json' || media.startsWith('text/') || /\.(json|txt|md|csv)$/i.test(name)) return 'text'
  return 'download'
}
function downloadBlob(blob, fileName) {
  const url = URL.createObjectURL(blob)
  const anchor = document.createElement('a')
  anchor.href = url
  anchor.download = fileName || 'download.bin'
  anchor.click()
  URL.revokeObjectURL(url)
}
async function showBlob(blob, ref, pageNo) {
  const kind = ref.downloadOnly ? 'download' : mediaKind(ref.mediaType, ref.artifactName)
  if (kind === 'download') return downloadBlob(blob, ref.artifactName)
  clearPreview()
  preview.visible = true
  preview.title = ref.artifactName || '证据产物'
  preview.kind = kind
  if (kind === 'text') preview.text = await blob.text()
  else {
    preview.url = URL.createObjectURL(blob)
    if (kind === 'pdf' && pageNo) preview.url += `#page=${pageNo}`
  }
}
async function previewArtifact(ref, evidence) {
  if (!ref?.artifactId) return
  preview.loading = true
  try {
    const blob = await getJobArtifactContent(jobId.value, ref.artifactId)
    await showBlob(blob, ref, evidence?.locator?.pageNo)
  } finally { preview.loading = false }
}
function clearPreview() {
  if (preview.url) URL.revokeObjectURL(preview.url.split('#')[0])
  Object.assign(preview, { loading: false, kind: '', url: '', text: '', title: '' })
}
function schedulePoll() {
  clearTimeout(pollTimer)
  if (disposed) return
  const active = workflows.value.some(workflow => (
    isProcessing(executionState(workflow)) || isProcessing(analysisState(workflow))
  )) || isProcessing(reportState.value)
  if (active) pollTimer = setTimeout(() => loadEvidence(true), 3000)
}
async function loadEvidence(silent = false) {
  const currentJobId = jobId.value
  if (!currentJobId) return
  if (!silent) loading.value = true
  const sequence = ++loadSequence
  if (!silent) {
    // 切换任务/审查项时先清空旧数据，避免残留上一条证据链
    workflows.value = []
    report.value = {}
    reportState.value = { status: 'PENDING', errorMessage: null }
    jobStatus.value = ''
  }
  try {
    const params = {}
    if (workflowRunId.value) params.workflowRunId = workflowRunId.value
    const response = await listJobEvidenceChain(currentJobId, params)
    if (sequence !== loadSequence || currentJobId !== jobId.value) return
    const data = response.data || {}
    workflows.value = (data.workflows || []).map(workflow => ({ ...workflow, analysisView: analysisView(workflow) }))
    report.value = data.report || {}
    reportState.value = data.reportProcessingState || { status: report.value.reportId ? 'SUCCESS' : 'MISSING' }
    jobStatus.value = data.jobStatus || ''
  } finally {
    if (sequence === loadSequence) {
      if (!silent) loading.value = false
      schedulePoll()
    }
  }
}
function refreshOnFocus() {
  if (!disposed) loadEvidence(true)
}
onMounted(() => {
  window.addEventListener('focus', refreshOnFocus)
  loadEvidence()
})
// 从消息中心跳到另一个审查项时路由 path 不变，只换 query，需要重新加载
watch([jobId, workflowRunId], () => {
  clearTimeout(pollTimer)
  loadEvidence()
})
onBeforeUnmount(() => {
  disposed = true
  window.removeEventListener('focus', refreshOnFocus)
  clearTimeout(pollTimer)
  if (claimHighlightTimer) {
    clearTimeout(claimHighlightTimer)
    claimHighlightTimer = null
  }
  clearPreview()
})
</script>

<style>
.evidence-chain-page { padding: 16px; }
.page-header { margin-bottom: 16px; }
.workflow-card { margin-bottom: 18px; padding: 16px; border: 1px solid var(--app-border, #dcdfe6); border-radius: 8px; background: var(--app-surface, #fff); }
.workflow-header, .claim-header, .evidence-card-head { display: flex; align-items: center; justify-content: space-between; gap: 12px; }
.workflow-header { margin-bottom: 14px; }
.workflow-header h3 { margin: 0 0 5px; }
.workflow-header span, .claim-header code { color: var(--app-text, #909399); font-size: 12px; }
.workflow-tags { display: flex; flex-wrap: wrap; justify-content: flex-end; gap: 6px; }
.processing-block { position: relative; min-height: 72px; }
.processing-block + .processing-block { margin-top: 14px; }
.claims-section { scroll-margin-top: 24px; }
.claim-card {
  scroll-margin-top: 90px;
  transition: background-color .25s ease, border-color .25s ease, box-shadow .25s ease;
}
.claim-card.is-claim-highlighted {
  border-color: var(--el-color-warning, #e6a23c);
  background: var(--el-color-warning-light-9, #fdf6ec);
  box-shadow: 0 0 0 3px var(--el-color-warning-light-5, #f3d19e);
}
.review-jump-link {
  display: inline-flex;
  vertical-align: baseline;
  font-size: inherit;
  font-weight: 600;
}
.review-jump-link:focus-visible {
  outline: 2px solid var(--el-color-warning);
  outline-offset: 3px;
  border-radius: 3px;
}
@media (prefers-reduced-motion: reduce) {
  .claim-card { transition: none; }
}
.processing-mask {
  position: absolute;
  z-index: 3;
  inset: -2px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 8px;
  color: var(--app-text, #303133);
  background: rgb(255 255 255 / 88%);
  backdrop-filter: blur(1.5px);
  border-radius: 7px;
}
.processing-mask small { color: var(--app-text, #737373); font-size: 12px; letter-spacing: 0.04em; }
.processing-ring {
  width: 22px;
  height: 22px;
  box-sizing: border-box;
  border: 2px solid var(--app-border, #d4d4d4);
  border-top-color: var(--app-border, #171717);
  border-radius: 50%;
  animation: evidenceProcessingSpin 0.8s linear infinite;
}
@keyframes evidenceProcessingSpin { to { transform: rotate(360deg); } }
@media (prefers-reduced-motion: reduce) {
  .processing-ring { animation: none; }
}
.section-title { margin: 18px 0 10px; font-weight: 600; }
.tool-summary-heading { display: flex; align-items: center; justify-content: space-between; gap: 12px; }
.tool-summary-toggle {
  display: inline-flex; align-items: center; justify-content: center; flex-shrink: 0;
  width: 30px; height: 30px; padding: 0; border: 0; border-radius: 8px;
  background: transparent; color: var(--el-text-color-primary); cursor: pointer;
}
.tool-summary-toggle:hover { background: var(--el-fill-color-light); }
.tool-summary-toggle:focus-visible { outline: 2px solid var(--el-color-primary); outline-offset: 2px; }
.tool-summary-toggle .el-icon { transition: transform .2s ease; }
.tool-summary-toggle.is-expanded .el-icon { transform: rotate(180deg); }
@media (prefers-reduced-motion: reduce) {
  .tool-summary-toggle .el-icon { transition: none; }
}

.claim-card { margin-top: 10px; padding: 13px; border: 1px solid var(--app-border, #ebeef5); border-radius: 6px; background: var(--app-surface, #fafafa); }
.claim-header { margin-bottom: 10px; }
.claim-header div > * { display: block; }
.claim-header code { margin-top: 4px; }
.evidence-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(330px, 1fr)); gap: 10px; }
.evidence-card { min-width: 0; padding: 12px; border: 1px solid var(--app-border, #dcdfe6); border-radius: 6px; background: var(--app-surface, #fff); }
.evidence-card-head { flex-wrap: wrap; }
.evidence-card-head > strong { min-width: 0; overflow-wrap: anywhere; }
.evidence-card-tags { display: flex; flex-wrap: wrap; align-items: center; gap: 6px; margin-left: auto; }
.evidence-match-status { padding: 3px 8px; border: 1px solid currentColor; border-radius: 4px; color: var(--evidence-status-text); font-size: 12px; font-weight: 600; white-space: nowrap; }
.evidence-card.is-performance-match,
.evidence-card.is-validity-valid,
.evidence-card.is-seal-signature-passed,
.evidence-card.is-evidence-present {
  --evidence-status-text: var(--app-success-text, #237a46);
  --evidence-status-accent: #42c987;
  --evidence-status-surface: #183d2c;
  background: linear-gradient(90deg, #edf9f1 0%, rgb(237 249 241 / 72%) 14%, rgb(237 249 241 / 0%) 33%), var(--app-surface, #fff);
}
.evidence-card.is-performance-mismatch,
.evidence-card.is-validity-expired,
.evidence-card.is-seal-signature-failed,
.evidence-card.is-evidence-missing {
  --evidence-status-text: var(--app-danger-text, #b93838);
  --evidence-status-accent: #ff7878;
  --evidence-status-surface: #48252b;
  background: linear-gradient(90deg, #fdf0f0 0%, rgb(253 240 240 / 72%) 14%, rgb(253 240 240 / 0%) 33%), var(--app-surface, #fff);
}
html.dark .evidence-card:is(.is-performance-match, .is-performance-mismatch, .is-validity-valid, .is-validity-expired, .is-seal-signature-passed, .is-seal-signature-failed, .is-evidence-present, .is-evidence-missing) {
  border-color: var(--evidence-status-accent);
  box-shadow: inset 4px 0 0 var(--evidence-status-accent);
  background: var(--evidence-status-surface);
}
.evidence-legend { margin: 0 0 10px; color: var(--app-text, #737373); font-size: 12px; }
.locator-grid { display: grid; grid-template-columns: 76px minmax(0, 1fr); gap: 5px 8px; margin: 10px 0; font-size: 12px; }
.locator-grid dt { color: var(--app-text, #909399); }
.locator-grid dd { min-width: 0; margin: 0; overflow-wrap: anywhere; }
.evidence-snippet { max-height: 150px; margin: 8px 0; padding: 9px; overflow: auto; white-space: pre-wrap; word-break: break-word; color: var(--app-primary-text, #475569); background: var(--app-surface-muted, #f5f7fa); border-radius: 4px; }
.artifact-actions { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 10px; }
.artifact-actions .el-button + .el-button { margin-left: 0; }
.diagnostic-row { margin-bottom: 8px; }
.analysis-section, .failure-diagnostics { margin-top: 14px; }
.tool-summary-section, .analysis-section, .failure-diagnostics { padding: 0 2px 2px; }
.analysis-paper { margin-top: 10px; padding: 14px; border: 1px solid var(--app-border, #ebeef5); border-radius: 6px; background: var(--app-surface, #fff); }
.analysis-field { margin-bottom: 14px; }
.analysis-field:last-child { margin-bottom: 0; }
.analysis-field h4 { margin: 0 0 6px; font-size: 14px; color: var(--app-text, #303133); }
.analysis-groups { margin-top: 12px; }
.analysis-group + .analysis-group { margin-top: 12px; }
.analysis-group-title { margin: 0 0 6px; padding-left: 8px; font-size: 16px; color: var(--app-primary-text, #409eff); border-left: 3px solid var(--app-primary-border, #409eff); font-weight: bold;}
.analysis-group-body { padding-left: 40px; }
.analysis-text { margin: 0 0 6px; color: var(--app-text, #4b5563); line-height: 1.75; white-space: pre-line; }
.analysis-tone-warning { color: var(--app-warning-text, #e6a23c); }
.analysis-tone-success { color: var(--app-success-text, #67c23a); }
.analysis-list { margin: 0 0 6px; padding-left: 12px; }
.analysis-list li { margin: 2px 0; line-height: 1.75; color: var(--app-text, #4b5563); }
.analysis-sublist { margin: 2px 0 0 20px; list-style: circle; }
.report-section { margin-top: 18px; padding: 16px; border: 1px solid var(--app-border, #dcdfe6); border-radius: 8px; background: var(--app-surface-muted, #f3f5f8); }
.report-header { display: flex; align-items: center; justify-content: space-between; gap: 12px; margin-bottom: 12px; }
.report-header h3 { margin: 0 0 5px; }
.report-header span { color: var(--app-text, #909399); font-size: 12px; }
.report-paper { width: min(980px, 100%); box-sizing: border-box; margin: 14px auto 0; padding: 32px 40px; background: var(--app-surface, #fff); box-shadow: 0 6px 20px rgb(15 23 42 / 8%); }
.preview-body { min-height: 240px; }
.preview-image { display: block; width: 100%; height: 70vh; }
.preview-pdf { width: 100%; height: 72vh; border: 0; }
.preview-text { max-height: 70vh; margin: 0; padding: 14px; overflow: auto; white-space: pre-wrap; word-break: break-word; background: #111827; color: #e5e7eb; border-radius: 6px; }
.text-highlight {
  background-color: var(--app-danger-surface, #fde2e2);
  color: var(--app-danger-text, #d93025);
  padding: 2px 4px;
  border-radius: 3px;
  font-weight: normal;
}
</style>

下面是analysis.js文件：
// 证据链页面的单项分析 markdown 由后端按“标题 + 平铺列表”拼出，字段值里还会夹着空行和子级列表，
// 这里按已知字段名切成结构化数据，页面只展示：结论 / 证据详情 / 人工决定 / 复核备注 / 重试建议。

const FIELD_LABELS = ['业务结论', '系统原结论', '人工决定', '复核备注', '证据解释', '失败根因', '重试建议', '建议置信度']
const FIELD_PATTERN = new RegExp(`^[-*]\\s*(${FIELD_LABELS.join('|')})[：:]\\s*(.*)$`)
const SUGGESTION_HEADER = '改进建议：'
const SUB_TASK_START = /子任务\s*\d*\s*[（(]/

export function splitAnalysisFields(markdown) {
  const buffers = {}
  let current = null
  for (const rawLine of String(markdown || '').split('\n')) {
    const line = rawLine.trim()
    if (line.startsWith('###') || line.startsWith('|') || line === SUGGESTION_HEADER) {
      // 大标题、证据表格、改进建议都不再展示，遇到即结束当前字段
      current = null
      continue
    }
    const matched = FIELD_PATTERN.exec(line)
    if (matched) {
      current = matched[1]
      buffers[current] = [matched[2].trim()]
      continue
    }
    if (current) buffers[current].push(line)
  }
  return Object.fromEntries(
    Object.entries(buffers).map(([label, lines]) => [
      label,
      lines.join('\n').replace(/\n{3,}/g, '\n\n').trim()
    ])
  )
}

// 结论里的子任务名可能带嵌套括号，如“子任务1（法定代表人（单位负责人）授权委托书）”
export function splitSubTaskHeading(paragraph) {
  const start = paragraph.search(SUB_TASK_START)
  if (start < 0) return null
  const openIndex = start + paragraph.slice(start).search(/[（(]/)
  const openChar = paragraph[openIndex]
  const closeChar = openChar === '（' ? '）' : ')'
  let depth = 0
  let endIndex = -1
  for (let index = openIndex; index < paragraph.length; index += 1) {
    if (paragraph[index] === openChar) depth += 1
    else if (paragraph[index] === closeChar) {
      depth -= 1
      if (depth === 0) {
        endIndex = index
        break
      }
    }
  }
  if (endIndex < 0) return null
  const number = paragraph.slice(start, openIndex).replace(/[^0-9]/g, '')
  const name = paragraph.slice(openIndex + 1, endIndex).trim()
  return {
    heading: `子任务${number || ''}（${name}）`,
    text: paragraph.slice(endIndex + 1).replace(/^[：:]\s*/, '').trim()
  }
}

export function buildDetailBlocks(text) {
  const blocks = []
  let paragraphLines = []
  let listItems = []
  let listType = 'list'
  const flushParagraph = () => {
    if (!paragraphLines.length) return
    const paragraph = paragraphLines.join('\n')
    const subTask = splitSubTaskHeading(paragraph)
    blocks.push(subTask
      ? { type: 'text', heading: subTask.heading, text: subTask.text }
      : { type: 'text', heading: '', text: paragraph })
    paragraphLines = []
  }
  const flushList = () => {
    if (listItems.length) blocks.push({ type: listType, items: listItems })
    listItems = []
    listType = 'list'
  }
  for (const rawBlock of String(text || '').replace(/\r/g, '').trim().split(/\n{2,}/)) {
    for (const rawLine of rawBlock.split('\n')) {
      const line = rawLine.trim()
      if (!line) continue
      // 缩进的“- ”是上一级条目的下一级列表（后端结论用两空格缩进表示）
      const bullet = rawLine.match(/^(\s*)-\s+(.*)$/)
      if (bullet) {
        flushParagraph()
        const type = bullet[1] ? 'sublist' : 'list'
        if (type !== listType) flushList()
        listType = type
        listItems.push(bullet[2].trim())
      } else {
        flushList()
        paragraphLines.push(line)
      }
    }
    flushParagraph()
    flushList()
  }
  return blocks
}

// 4.3/4.4 会按商务文件、技术文件各跑一次：缺少对应文件的子任务会被后端跳过，
// 只产出一条不带证据的 Claim（“已确认审查所需输入缺失或无法定位”），事实主张里不再展示这类空卡片。
export function visibleClaims(workflow = {}) {
  return (workflow.claims || []).filter(claim => Boolean(claim?.evidence?.length))
}

// 3.1/4.1/4.2 的证据卡片由后端 meta.card_state 标注是否取到证据（present/missing），
// 页面据此给卡片标绿/标红，配色沿用 4.5 的卡片设计（is-performance-match / is-performance-mismatch）。
export function evidenceCardState(evidence = {}) {
  const state = String(evidence?.meta?.card_state || '').toLowerCase()
  return state === 'present' || state === 'missing' ? state : ''
}

const EVIDENCE_COLOR_LEGENDS = {
  gsxt_screenshot_check: '注意：绿色背景表示报告生成日期符合时效，红色背景表示报告生成日期不符合时效或未获取到证据。',
  auth_letter_ledger_compare: '注意：绿色背景表示已获取图片或文本证据，且字段一致；红色背景表示未获取图片或文本证据，或虽已获取文本证据但字段不一致。',
  tech_doc_field_compare: '注意：绿色背景表示已获取文本证据且字段一致；红色背景表示未获取到文本证据，或已获取文本证据但字段不一致。',
  seal_signature_check: '注意：绿色背景表示获取到证据，红色背景表示未获取到证据。',
  certificate_validity_check: '注意：绿色背景表示获取到证据或者有效期限有效，红色背景表示未获取到证据、有效期限无效，或汇总表的有效期与证书有效期不一致。',
  performance_screenshot_check: '注意：绿色背景表示获取到证据，红色背景表示未获取到证据。',
}

// 事实主张下方的红绿背景说明，只有需要上色的审查项才展示
export function evidenceColorLegend(workflowCode) {
  return EVIDENCE_COLOR_LEGENDS[String(workflowCode || '')] || ''
}

export function analysisView(workflow = {}) {
  const fields = splitAnalysisFields(workflow.analysis?.markdownContent)
  // 4.3/4.4 会按商务文件、技术文件各执行一次，分析记录里的“系统原结论”是多次结论用 “ | ” 拼起来的；
  // 每张证据链卡片只展示本次 workflow run 自己的结论，run 结论缺失时才退回合并文本
  const runConclusion = String(workflow.conclusion || '').trim()
  const evidenceDetail = buildDetailBlocks(runConclusion || fields['系统原结论'])
  const conclusion = fields['业务结论'] || ''
  return {
    conclusion,
    conclusionParts: splitToneParts(conclusion),
    evidenceDetail,
    evidenceGroups: groupAnalysisBlocks(evidenceDetail),
    humanDecision: fields['人工决定'] || '未提交',
    reviewComment: fields['复核备注'] || '无',
    retryRecommendation: fields['重试建议'] || '-'
  }
}

// heading 表示章节标题，其后的内容都归到该章节下（页面按此缩进展示）
export function groupAnalysisBlocks(blocks) {
  const groups = []
  for (const block of blocks || []) {
    const heading = String(block?.heading || '').trim()
    if (heading || !groups.length) groups.push({ heading, blocks: [] })
    const group = groups[groups.length - 1]
    if (block?.type === 'list' || block?.type === 'sublist') {
      if (block.items?.length) group.blocks.push({ type: block.type, items: block.items.map(splitLabelParts) })
    } else if (block?.text) {
      // 先按换行拆行再解析标签；【】里的换行同样不能拆，否则后半段没有成对的【】会丢失背景
      group.blocks.push({
        type: 'text',
        parts: splitOutsideBrackets(block.text, char => char === '\n')
          .flatMap(piece => (piece === '\n' ? [{ break: true }] : splitLabelParts(piece)))
      })
    }
  }
  return groups
}

// 冒号前的短前缀（如“项目名称：”“核查结果如下：”）渲染时加粗；
// 标签必须是行首或紧跟“，。、；！？”等分隔符之后，避免把“需人工复核。核查结果如下：”整段吞成标签
const LABEL_SEPARATORS = '；;，,。！？!?'
const LABEL_SEPARATOR_ONLY = new RegExp(`^[${LABEL_SEPARATORS}]$`)

// 按分隔符切分，但【】里的内容整体保留（值里可能带“，”和换行，
// 一旦被切开，后半段就没有成对的【】，页面不会给这段文字加背景）。
export function splitOutsideBrackets(text, isDelimiter) {
  const source = String(text || '')
  const pieces = []
  let buffer = ''
  let depth = 0
  for (let index = 0; index < source.length; index += 1) {
    const char = source[index]
    // 只有存在配对“】”时才进入括号内，避免落单的“【”把后面的标签切分也吞掉
    if (char === '【' && source.indexOf('】', index) > index) depth += 1
    else if (char === '】' && depth > 0) depth -= 1
    if (depth === 0 && isDelimiter(char)) {
      if (buffer) pieces.push(buffer)
      pieces.push(char)
      buffer = ''
      continue
    }
    buffer += char
  }
  if (buffer) pieces.push(buffer)
  return pieces
}

export function splitLabelParts(line) {
  const parts = []
  for (const segment of splitOutsideBrackets(line, char => LABEL_SEPARATOR_ONLY.test(char))) {
    if (!segment) continue
    if (LABEL_SEPARATOR_ONLY.test(segment)) {
      parts.push({ text: segment, bold: false, tone: '' })
      continue
    }
    const matched = segment.match(/^([^：:]{1,80})([：:])\s*(.*)$/)
    if (matched) {
      parts.push({ text: `${matched[1]}${matched[2]}`, bold: true, tone: '' })
      parts.push(...splitToneParts(matched[3]))
    } else {
      parts.push(...splitToneParts(segment))
    }
  }
  return parts
}

const WARNING_WORDS = ['需人工复核', '需复核', '审查不通过', '审核不通过', '检测不通过', '校验不通过']
const SUCCESS_WORDS = ['均通过', '审查通过', '审核通过', '检测通过', '校验通过']
const TONE_WORDS = new Set([...WARNING_WORDS, ...SUCCESS_WORDS])
const TONE_PATTERN = new RegExp(`(${[...TONE_WORDS].join('|')})`, 'g')

const BRACKET_PATTERN = /(【[^【】]*】)/

// 结论性关键词单独成段，页面分别用黄色（需复核/不通过）和绿色（通过）显示；
// 【】中的内容进行红色背景强调，但不加粗
export function splitToneParts(text) {
  const result = []

  for (const segment of String(text || '').split(TONE_PATTERN)) {
    if (!segment) continue

    const tone = TONE_WORDS.has(segment)
      ? (WARNING_WORDS.includes(segment) ? 'warning' : 'success')
      : ''

    for (const piece of segment.split(BRACKET_PATTERN)) {
      if (!piece) continue

      // 判断是否为完整的【...】
      const match = piece.match(/^【([^【】]*)】$/)

      if (match) {
        // 【】符号单独输出，不设置 highlight
        result.push({text: '【', bold: false, tone,})

        // 只有【】里面的内容设置 highlight
        result.push({text: match[1], bold: false, tone, highlight: true,})

        // 】符号单独输出，不设置 highlight
        result.push({text: '】', bold: false, tone,})
      } else {
        result.push({text: piece, bold: false, tone,})
      }
    }
  }
  return result
}

