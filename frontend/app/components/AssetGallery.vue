<template>
  <div class="assets-wrap">
    <div class="asset-toolbar">
      <div class="seg asset-filter" role="tablist" :aria-label="t('detail.tabs.assets')">
        <button
          v-for="tab in assetTabs"
          :key="tab.value"
          type="button"
          role="tab"
          :aria-selected="assetTab === tab.value"
          class="seg-item"
          :class="{ on: assetTab === tab.value }"
          @click="assetTab = tab.value"
        >
          <span>{{ tab.label }}</span>
          <span class="seg-count mono">{{ tab.count }}</span>
        </button>
      </div>
    </div>

    <!-- 全部素材为空 -->
    <div v-if="!materials.length" class="empty-state">
      <div class="empty-icon">
        <ImageIcon :size="26" :stroke-width="1.4" />
      </div>
      <p class="empty-title">{{ t('detail.assets.emptyTitle') }}</p>
      <p class="empty-desc">{{ t('detail.assets.emptyDesc') }}</p>
    </div>

    <div v-else class="asset-groups">
      <template v-for="g in assetGroups" :key="g.kindKey">
        <section v-if="g.items.length" class="asset-section">
          <div v-if="assetTab === 'all'" class="asset-group-head" :class="tagClass(g.kindKey)">
            <span class="group-icon">
              <User v-if="g.kindKey === 'character'" :size="14" :stroke-width="2" />
              <MapPin v-else-if="g.kindKey === 'scene'" :size="14" :stroke-width="2" />
              <Box v-else :size="14" :stroke-width="2" />
            </span>
            <span class="group-label">{{ g.label }}</span>
            <span class="group-count mono">{{ g.items.length }}</span>
          </div>

          <!-- 角色卡片网格 -->
          <div v-if="g.kindKey === 'character'" class="character-asset-grid">
            <article
              v-for="m in g.items"
              :key="'character-' + m.id"
              class="card character-asset-card"
              tabindex="0"
              role="button"
              @click="openEdit(m)"
              @keydown.enter.self.prevent="openEdit(m)"
              @keydown.space.self.prevent="openEdit(m)"
            >
              <div class="asset-cover wide">
                <img
                  v-if="matHasImage(m)"
                  :src="thumbOf(assetSrc(m))"
                  class="previewable-image"
                  loading="lazy"
                  @error="thumbFallback($event, assetSrc(m))"
                  @click.stop="openAssetViewer(m)"
                />
                <div v-else class="asset-cover-empty">
                  <User :size="22" :stroke-width="1.4" />
                </div>
                <span class="asset-cover-badge" :class="matHasImage(m) ? 'is-ready' : (isPending(m) ? 'is-pending' : '')">
                  {{ matHasImage(m) ? t('episode.asset.portraitReady') : (isPending(m) ? t('episode.asset.portraitPending') : t('episode.asset.portraitTodo')) }}
                </span>
              </div>

              <div class="character-body">
                <div class="character-name-row">
                  <strong class="character-name truncate">{{ m.name }}</strong>
                  <span class="tag">{{ m.role || t('common.role') }}</span>
                </div>
                <div class="character-visual-summary" :title="matDesc(m)">
                  <span class="truncate">{{ t('episode.asset.appearance') }}{{ m.appearance || t('detail.assets.todoShort') }}</span>
                  <span class="truncate">{{ t('episode.asset.styling') }}{{ m.styling || t('detail.assets.todoShort') }}</span>
                </div>
                <div class="asset-final-prompt" :title="matFinalPrompt(m)">
                  <span class="afp-label">{{ t('episode.asset.finalPromptTurnaround') }}</span>
                  <span :class="['afp-text', !matFinalPrompt(m) && 'dim']">
                    {{ matFinalPrompt(m) || t('episode.asset.finalPromptAutoTurnaround') }}
                  </span>
                </div>
                <CharacterVoice :character="m" @updated="onCharacterVoiceUpdated" />
              </div>

              <div class="asset-foot">
                <span :class="['dot', matHasImage(m) && 'ok', isPending(m) && 'pending']" />
                <button
                  class="btn btn-sm ml-auto"
                  type="button"
                  :title="t('episode.asset.uploadCharImage')"
                  :disabled="isUploading(m)"
                  @click.stop="uploadMaterial(m)"
                >
                  <Loader2 v-if="isUploading(m)" :size="12" class="animate-spin" />
                  <Upload v-else :size="12" />
                  {{ t('episode.asset.upload') }}
                </button>
                <button
                  class="btn btn-sm"
                  type="button"
                  :disabled="isPending(m)"
                  @click.stop="generateMaterial(m)"
                >
                  <Loader2 v-if="isPending(m)" :size="12" class="animate-spin" />
                  {{ matHasImage(m) ? t('episode.asset.regen') : (isPending(m) ? t('episode.asset.generating') : t('episode.asset.generate')) }}
                </button>
              </div>
            </article>
          </div>

          <!-- 场景 / 道具卡片网格 -->
          <div v-else class="asset-grid">
            <article
              v-for="m in g.items"
              :key="g.kindKey + '-' + m.id"
              :class="['card', 'asset-card', 'asset-click-card', g.kindKey === 'prop' ? 'prop-card' : '']"
              tabindex="0"
              role="button"
              @click="openEdit(m)"
              @keydown.enter.prevent="openEdit(m)"
              @keydown.space.prevent="openEdit(m)"
            >
              <div class="asset-cover wide">
                <img
                  v-if="matHasImage(m)"
                  :src="thumbOf(assetSrc(m))"
                  class="previewable-image"
                  loading="lazy"
                  @error="thumbFallback($event, assetSrc(m))"
                  @click.stop="openAssetViewer(m)"
                />
                <div v-else class="asset-cover-empty">
                  <MapPin v-if="g.kindKey === 'scene'" :size="22" :stroke-width="1.4" />
                  <Box v-else :size="22" :stroke-width="1.4" />
                </div>
                <span class="asset-cover-badge" :class="matHasImage(m) ? 'is-ready' : (isPending(m) ? 'is-pending' : '')">
                  {{ matHasImage(m) ? t('episode.asset.ready') : (isPending(m) ? t('episode.asset.generating') : t('episode.asset.todo')) }}
                </span>
              </div>

              <div class="asset-body">
                <template v-if="g.kindKey === 'scene'">
                  <div class="asset-name" :title="m.location">{{ m.location }}</div>
                  <div class="asset-meta asset-desc dim" :title="matDesc(m)">{{ matDesc(m) || t('episode.asset.sceneDescTodo') }}</div>
                  <div v-if="m.lighting" class="asset-meta asset-light dim" :title="m.lighting">{{ t('episode.asset.lighting') }}{{ m.lighting }}</div>
                </template>
                <template v-else>
                  <div class="prop-name-row">
                    <span class="asset-name" :title="m.name">{{ m.name }}</span>
                    <span class="tag">{{ m.type || t('common.prop') }}</span>
                  </div>
                  <div class="asset-meta asset-desc dim" :title="m.description || ''">{{ m.description || t('episode.asset.noDescription') }}</div>
                </template>
                <div class="asset-meta asset-final" :class="{ dim: !matFinalPrompt(m) }" :title="matFinalPrompt(m)">
                  <span class="afp-label">{{ g.kindKey === 'scene' ? t('episode.asset.finalPromptFixed') : t('episode.asset.finalPromptWhiteBg') }}</span>
                  {{ matFinalPrompt(m) || (g.kindKey === 'scene' ? t('episode.asset.finalPromptAutoFixed') : t('episode.asset.finalPromptAutoWhiteBg')) }}
                </div>
              </div>

              <div class="asset-foot">
                <span :class="['dot', matHasImage(m) && 'ok', isPending(m) && 'pending']" />
                <button
                  class="btn btn-sm ml-auto"
                  type="button"
                  :title="t('episode.asset.uploadImage')"
                  :disabled="isUploading(m)"
                  @click.stop="uploadMaterial(m)"
                >
                  <Loader2 v-if="isUploading(m)" :size="12" class="animate-spin" />
                  <Upload v-else :size="12" />
                  {{ t('episode.asset.upload') }}
                </button>
                <button
                  class="btn btn-sm"
                  type="button"
                  :disabled="isPending(m)"
                  @click.stop="generateMaterial(m)"
                >
                  <Loader2 v-if="isPending(m)" :size="12" class="animate-spin" />
                  {{ matHasImage(m) ? t('episode.asset.regen') : (isPending(m) ? t('episode.asset.generating') : t('episode.asset.generate')) }}
                </button>
              </div>
            </article>
          </div>
        </section>
      </template>

      <!-- 筛选某一类但该类暂无素材 -->
      <div v-if="assetTab !== 'all' && !visibleAssets.length" class="empty-state">
        <div class="empty-icon">
          <ImageIcon :size="26" :stroke-width="1.4" />
        </div>
        <p class="empty-title">{{ t('detail.assets.emptyKindTitle', { kind: tabLabel(assetTab) }) }}</p>
        <p class="empty-desc">{{ t('detail.assets.emptyKindDesc', { kind: tabLabel(assetTab) }) }}</p>
      </div>
    </div>

    <!-- 素材详情 / 编辑对话框 -->
    <Teleport to="body">
      <div v-if="editDialog && editTarget" class="overlay mat-detail-overlay" @click.self="closeEdit">
        <section class="dialog mat-detail-dialog" :aria-label="t('detail.mat.dialogAria')">
          <header class="dialog-head mat-detail-head">
            <div class="mat-detail-title-block">
              <span class="eyebrow mat-detail-kicker">
                {{ editTarget.kindKey === 'character' ? t('episode.asset.typeChar') : editTarget.kindKey === 'scene' ? t('episode.asset.typeScene') : t('episode.asset.typeProp') }}
              </span>
              <h2 class="mat-detail-title">{{ editTarget.name || editTarget.location || t('detail.mat.unnamed') }}</h2>
            </div>
            <div class="mat-detail-head-actions">
              <span v-if="editTarget.kindKey === 'character'" class="tag">{{ editTarget.role || t('common.role') }}</span>
              <span v-else-if="editTarget.kindKey === 'prop'" class="tag">{{ editTarget.type || t('common.prop') }}</span>
              <span v-else class="tag">{{ editTarget.time || t('episode.asset.noTime') }}</span>
              <button type="button" class="btn btn-ghost btn-icon" :aria-label="t('common.close')" @click="closeEdit">
                <X :size="15" />
              </button>
            </div>
          </header>

          <div class="dialog-body mat-detail-body">
            <div class="mat-detail-shell">
              <!-- 左侧：视觉预览 -->
              <aside class="mat-detail-preview-panel">
                <div class="mat-detail-section-title">
                  <span>{{ t('episode.asset.visualPreview') }}</span>
                  <span :class="['mat-detail-state', matHasImage(editTarget) ? 'is-ready' : '']">
                    {{ matHasImage(editTarget) ? t('episode.asset.ready') : t('episode.asset.todo') }}
                  </span>
                </div>

                <button
                  type="button"
                  class="mat-detail-media-frame"
                  :disabled="!matHasImage(editTarget)"
                  @click.stop="openAssetViewer(editTarget)"
                >
                  <img
                    v-if="matHasImage(editTarget)"
                    :src="thumbOf(assetSrc(editTarget))"
                    @error="thumbFallback($event, assetSrc(editTarget))"
                  />
                  <span v-else class="mat-detail-media-empty">
                    <User v-if="editTarget.kindKey === 'character'" :size="28" :stroke-width="1.3" />
                    <Box v-else-if="editTarget.kindKey === 'prop'" :size="28" :stroke-width="1.3" />
                    <MapPin v-else :size="28" :stroke-width="1.3" />
                  </span>
                </button>

                <div class="mat-detail-meta-row">
                  <div class="mat-detail-meta-item">
                    <span>{{ t('episode.asset.kindLabel') }}</span>
                    <strong>{{ editTarget.kindKey === 'character' ? t('episode.asset.charPortrait') : editTarget.kindKey === 'prop' ? t('common.prop') : t('episode.asset.sceneImage') }}</strong>
                  </div>
                  <div class="mat-detail-meta-item">
                    <span>{{ editTarget.kindKey === 'character' ? t('episode.asset.roleLabel') : editTarget.kindKey === 'prop' ? t('episode.asset.propTypeLabel') : t('episode.asset.timeLabel') }}</span>
                    <strong>{{ editTarget.kindKey === 'character' ? (editTarget.role || t('common.role')) : editTarget.kindKey === 'prop' ? (editTarget.type || t('common.prop')) : (editTarget.time || t('episode.asset.noTime')) }}</strong>
                  </div>
                </div>
                <CharacterVoice v-if="editTarget.kindKey === 'character'" :key="editTarget.id" :character="editTarget" @updated="onCharacterVoiceUpdated" />
              </aside>

              <!-- 右侧：编辑信息 -->
              <section class="mat-detail-editor-panel">
                <div class="mat-detail-section-title">
                  <span>{{ t('episode.asset.editInfo') }}</span>
                  <span class="dim">{{ editTarget.kindKey === 'character' ? t('episode.asset.editHintChar') : editTarget.kindKey === 'prop' ? t('episode.asset.editHintProp') : t('episode.asset.editHintScene') }}</span>
                </div>

                <!-- 道具：单列物品外貌 -->
                <div v-if="editTarget.kindKey === 'prop'" class="mat-detail-edit-grid mat-detail-edit-grid--prop">
                  <label class="mat-detail-edit-field">
                    <span>{{ t('episode.create.name') }}</span>
                    <input v-model="editDraft.name" class="input" :placeholder="t('episode.create.namePlaceholderProp')" />
                  </label>
                  <label class="mat-detail-edit-field">
                    <span>{{ t('episode.create.typeField') }}</span>
                    <input v-model="editDraft.type" class="input" :placeholder="t('detail.mat.propTypePlaceholder')" />
                  </label>
                  <label class="mat-detail-edit-field mat-detail-edit-field--full">
                    <span>{{ t('episode.asset.appearanceOfObject') }}</span>
                    <textarea v-model="editDraft.description" class="textarea mat-detail-textarea" rows="6" :placeholder="t('episode.asset.appearancePlaceholder')" />
                  </label>
                </div>

                <!-- 角色：样貌 + 妆造 -->
                <div v-else-if="editTarget.kindKey === 'character'" class="mat-detail-edit-grid mat-detail-edit-grid--character">
                  <div class="mat-detail-edit-pair">
                    <label class="mat-detail-edit-field">
                      <span>{{ t('episode.create.name') }}</span>
                      <input v-model="editDraft.name" class="input" :placeholder="t('episode.create.namePlaceholderChar')" />
                    </label>
                    <label class="mat-detail-edit-field">
                      <span>{{ t('episode.create.roleField') }}</span>
                      <input v-model="editDraft.role" class="input" :placeholder="t('episode.create.rolePlaceholder')" />
                    </label>
                  </div>
                  <label class="mat-detail-edit-field">
                    <span>{{ t('episode.asset.appearanceField') }}</span>
                    <textarea v-model="editDraft.appearance" class="textarea mat-detail-textarea" rows="5" :placeholder="t('episode.asset.appearanceFieldPlaceholder')" />
                  </label>
                  <label class="mat-detail-edit-field">
                    <span>{{ t('episode.asset.stylingField') }}</span>
                    <textarea v-model="editDraft.styling" class="textarea mat-detail-textarea" rows="5" :placeholder="t('episode.asset.stylingFieldPlaceholder')" />
                  </label>
                  <label class="mat-detail-edit-field">
                    <span>{{ t('detail.mat.personaField') }}</span>
                    <textarea v-model="editDraft.description" class="textarea mat-detail-textarea" rows="4" :placeholder="t('detail.mat.personaPlaceholder')" />
                  </label>
                </div>

                <!-- 场景：描述 + 光影 -->
                <div v-else class="mat-detail-edit-grid mat-detail-edit-grid--scene">
                  <div class="mat-detail-edit-pair">
                    <label class="mat-detail-edit-field">
                      <span>{{ t('episode.create.location') }}</span>
                      <input v-model="editDraft.location" class="input" :placeholder="t('detail.mat.locationPlaceholder')" />
                    </label>
                    <label class="mat-detail-edit-field">
                      <span>{{ t('episode.create.time') }}</span>
                      <input v-model="editDraft.time" class="input" :placeholder="t('detail.mat.timePlaceholder')" />
                    </label>
                  </div>
                  <label class="mat-detail-edit-field">
                    <span>{{ t('episode.asset.sceneDescField') }}</span>
                    <textarea v-model="editDraft.prompt" class="textarea mat-detail-textarea" rows="5" :placeholder="t('episode.asset.sceneDescPlaceholder')" />
                  </label>
                  <label class="mat-detail-edit-field">
                    <span>{{ t('episode.asset.sceneLightField') }}</span>
                    <textarea v-model="editDraft.lighting" class="textarea mat-detail-textarea" rows="5" :placeholder="t('episode.asset.sceneLightPlaceholder')" />
                  </label>
                </div>
              </section>
            </div>

            <!-- 最终提示词：可生成 / 重新生成 / 手动编辑 -->
            <section class="mat-detail-prompt-panel">
              <div class="mat-detail-section-title">
                <span>{{ t('detail.mat.finalPromptLabel') }}</span>
                <span class="dim">{{ t('detail.mat.finalPromptSub') }}</span>
                <button
                  type="button"
                  class="btn btn-sm mat-detail-prompt-gen"
                  :disabled="finalPromptGen || !firstEpisodeId"
                  :title="firstEpisodeId ? t('detail.mat.genPromptTitle') : t('detail.mat.needEpisodeFirst')"
                  @click="generateFinalPrompt(editTarget)"
                >
                  <Loader2 v-if="finalPromptGen" :size="12" class="animate-spin" />
                  <Sparkles v-else :size="12" />
                  {{ finalPromptGen ? t('episode.asset.generating') + '…' : (editDraft.finalPrompt ? t('episode.sb.regenPrompt') : t('episode.asset.genPrompt')) }}
                </button>
              </div>
              <textarea
                v-model="editDraft.finalPrompt"
                class="textarea mat-detail-prompt-text"
                rows="5"
                :placeholder="t('detail.mat.promptPlaceholder')"
              ></textarea>
            </section>
          </div>

          <footer class="dialog-foot mat-detail-foot">
            <div class="mat-detail-secondary-actions">
              <button type="button" class="btn" @click="closeEdit">{{ t('common.close') }}</button>
            </div>
            <div class="mat-detail-primary-actions">
              <button
                type="button"
                class="btn"
                :disabled="isUploading(editTarget)"
                @click="uploadMaterial(editTarget)"
              >
                <Loader2 v-if="isUploading(editTarget)" :size="12" class="animate-spin" />
                <Upload v-else :size="12" />
                {{ t('episode.asset.uploadImage') }}
              </button>
              <button
                type="button"
                class="btn"
                :disabled="isPending(editTarget)"
                @click="generateMaterial(editTarget)"
              >
                <Loader2 v-if="isPending(editTarget)" :size="12" class="animate-spin" />
                {{ matHasImage(editTarget) ? t('episode.asset.regen') : (isPending(editTarget) ? t('episode.asset.generating') + '…' : t('detail.mat.genImage')) }}
              </button>
              <button type="button" class="btn btn-primary" :disabled="editSaving" @click="saveEdit">
                <Loader2 v-if="editSaving" :size="12" class="animate-spin" />
                {{ editSaving ? t('detail.mat.saving') : t('episode.asset.saveChanges') }}
              </button>
            </div>
          </footer>
        </section>
      </div>
    </Teleport>

    <!-- 图片查看器 -->
    <Teleport to="body">
      <div v-if="assetViewer.open" class="overlay viewer-overlay" @click.self="closeAssetViewer">
        <div class="dialog viewer-dialog">
          <div class="viewer-head">
            <span class="viewer-title truncate">{{ assetViewer.title }}</span>
            <button type="button" class="btn btn-icon btn-sm btn-ghost" :aria-label="t('common.close')" @click="closeAssetViewer">
              <X :size="15" />
            </button>
          </div>
          <div class="viewer-stage">
            <img :src="assetViewer.src" :alt="assetViewer.title" class="viewer-img" />
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>

<script setup>
import { toast } from 'vue-sonner'
import { useI18n } from 'vue-i18n'
import { Box, Image as ImageIcon, Loader2, MapPin, Sparkles, Upload, User, X } from 'lucide-vue-next'
import { toastError } from '~/composables/useToast'
import { dramaAPI, characterAPI, sceneAPI, propAPI, uploadAPI } from '~/composables/useApi'
import { thumbOf, thumbFallback } from '~/composables/useMedia'
import CharacterVoice from '~/components/CharacterVoice.vue'

const props = defineProps({
  drama: { type: Object, required: true },
})
const emit = defineEmits(['refresh', 'voice-updated'])

const { t } = useI18n()

const assetTab = ref('all')
const assetViewer = ref({ open: false, src: '', title: '' })
const pendingMaterials = ref(new Set())
const uploadingMaterials = ref(new Set())
const KIND_ORDER = { character: 0, scene: 1, prop: 2 }

let unmounted = false
onUnmounted(() => { unmounted = true })

function matImage(m) { return m.image_url || m.imageUrl || m.localPath || m.local_path || '' }
function matHasImage(m) { return !!matImage(m) }
function matFinalPrompt(m) { return m.finalPrompt || m.final_prompt || '' }
function assetSrc(m) {
  const raw = matImage(m)
  if (!raw) return ''
  return /^https?:\/\//i.test(raw) || raw.startsWith('/') ? raw : `/${raw}`
}
function matDesc(m) {
  if (m.kindKey === 'character') return m.appearance || m.description || ''
  if (m.kindKey === 'scene') return m.prompt || m.description || ''
  return m.description || ''
}
function tagClass(kindKey) {
  return kindKey === 'character' ? 'is-character' : kindKey === 'scene' ? 'is-scene' : 'is-prop'
}

const materials = computed(() => {
  const d = props.drama
  if (!d) return []
  const list = []
  for (const c of d.characters || []) list.push({ ...c, kind: t('common.role'), kindKey: 'character' })
  for (const s of d.scenes || []) list.push({ ...s, kind: t('common.scene'), kindKey: 'scene' })
  for (const p of d.props || []) list.push({ ...p, kind: t('common.prop'), kindKey: 'prop' })
  return list.sort((a, b) => (KIND_ORDER[a.kindKey] - KIND_ORDER[b.kindKey]) || (a.id - b.id))
})

const assetTabs = computed(() => {
  const all = materials.value
  return [
    { label: t('index.status.all'), value: 'all', count: all.length },
    { label: t('common.role'), value: 'character', count: all.filter(m => m.kindKey === 'character').length },
    { label: t('common.scene'), value: 'scene', count: all.filter(m => m.kindKey === 'scene').length },
    { label: t('common.prop'), value: 'prop', count: all.filter(m => m.kindKey === 'prop').length },
  ]
})

function tabLabel(v) { return assetTabs.value.find(x => x.value === v)?.label || '' }

const visibleAssets = computed(() =>
  assetTab.value === 'all' ? materials.value : materials.value.filter(m => m.kindKey === assetTab.value),
)

const assetGroups = computed(() => {
  const groups = [
    { kindKey: 'character', label: t('common.role'), items: materials.value.filter(m => m.kindKey === 'character') },
    { kindKey: 'scene', label: t('common.scene'), items: materials.value.filter(m => m.kindKey === 'scene') },
    { kindKey: 'prop', label: t('common.prop'), items: materials.value.filter(m => m.kindKey === 'prop') },
  ]
  return assetTab.value === 'all'
    ? groups
    : groups.filter(g => g.kindKey === assetTab.value)
})

function pendingKey(m) { return `${m.kindKey}:${m.id}` }
function isPending(m) { return pendingMaterials.value.has(pendingKey(m)) }
function isUploading(m) { return uploadingMaterials.value.has(pendingKey(m)) }
function sleep(ms) { return new Promise(r => setTimeout(r, ms)) }

const firstEpisodeId = computed(() => props.drama?.episodes?.[0]?.id || null)

async function generateMaterial(m) {
  const epId = firstEpisodeId.value
  if (!epId) { toast.error(t('detail.mat.needEpisodeForImage')); return }
  const key = pendingKey(m)
  if (pendingMaterials.value.has(key)) return
  pendingMaterials.value = new Set(pendingMaterials.value).add(key)
  try {
    if (m.kindKey === 'character') await characterAPI.generateImage(m.id, epId)
    else if (m.kindKey === 'scene') await sceneAPI.generateImage(m.id, epId)
    else await propAPI.generateImage(m.id, epId)
    toast.success(t('detail.mat.generating', { kind: m.kind, name: m.name || m.location }))
    pollMaterial(m)
  } catch (e) {
    pendingMaterials.value = new Set([...pendingMaterials.value].filter(k => k !== key))
    toastError(e)
  }
}

async function pollMaterial(m) {
  const key = pendingKey(m)
  for (let i = 0; i < 40; i++) {
    await sleep(2500)
    if (unmounted) return
    try {
      const latest = await dramaAPI.get(props.drama.id)
      const list = m.kindKey === 'character' ? latest?.characters : m.kindKey === 'scene' ? latest?.scenes : latest?.props
      const rec = list?.find(x => x.id === m.id)
      if (rec && matImage(rec)) {
        pendingMaterials.value = new Set([...pendingMaterials.value].filter(k => k !== key))
        if (editTarget.value && editTarget.value.kindKey === m.kindKey && editTarget.value.id === m.id) {
          Object.assign(editTarget.value, rec)
        }
        emit('refresh')
        return
      }
    } catch { /* ignore transient poll errors */ }
  }
  if (unmounted) return
  pendingMaterials.value = new Set([...pendingMaterials.value].filter(k => k !== key))
  toast.info(t('detail.mat.genTimeout', { kind: m.kind, name: m.name || m.location }))
}

function uploadMaterial(m) {
  const key = pendingKey(m)
  if (uploadingMaterials.value.has(key)) return
  const input = document.createElement('input')
  input.type = 'file'
  input.accept = 'image/jpeg,image/png,image/webp,.jpg,.jpeg,.png,.webp'
  input.onchange = async () => {
    const file = input.files?.[0]
    if (!file) return
    uploadingMaterials.value = new Set(uploadingMaterials.value).add(key)
    try {
      const res = await uploadAPI.image(file)
      const payload = { image_url: res.path, local_path: res.path }
      if (m.kindKey === 'character') await characterAPI.update(m.id, payload)
      else if (m.kindKey === 'scene') await sceneAPI.update(m.id, payload)
      else await propAPI.update(m.id, payload)
      toast.success(t('detail.mat.uploaded', { kind: m.kind, name: m.name || m.location }))
      if (editTarget.value && editTarget.value.kindKey === m.kindKey && editTarget.value.id === m.id) {
        editTarget.value = { ...editTarget.value, image_url: res.path, local_path: res.path }
      }
      emit('refresh')
    } catch (e) {
      toastError(e)
    } finally {
      uploadingMaterials.value = new Set([...uploadingMaterials.value].filter(k => k !== key))
    }
  }
  input.click()
}

function openAssetViewer(m) {
  const titleName = m.name || m.location || t('detail.mat.unnamed')
  assetViewer.value = { open: true, src: assetSrc(m), title: `${m.kind} · ${titleName}` }
}
function closeAssetViewer() {
  assetViewer.value = { open: false, src: '', title: '' }
}

const editDialog = ref(false)
const editSaving = ref(false)
const editTarget = ref(null)
const editDraft = reactive({})

function onCharacterVoiceUpdated(id, voice) {
  const patch = { ...voice, voiceAudioUrl: voice.voice_audio_url, voiceAudioName: voice.voice_audio_name }
  if (editTarget.value?.kindKey === 'character' && editTarget.value.id === id) Object.assign(editTarget.value, patch)
  emit('voice-updated', id, voice)
}

function openEdit(m) {
  editTarget.value = { ...m }
  Object.keys(editDraft).forEach(k => delete editDraft[k])
  if (m.kindKey === 'character') {
    Object.assign(editDraft, { name: m.name || '', role: m.role || '', appearance: m.appearance || '', description: m.description || '', styling: m.styling || '' })
  } else if (m.kindKey === 'scene') {
    Object.assign(editDraft, { location: m.location || '', time: m.time || '', prompt: m.prompt || '', lighting: m.lighting || '' })
  } else {
    Object.assign(editDraft, { name: m.name || '', type: m.type || '', description: m.description || '' })
  }
  editDraft.finalPrompt = matFinalPrompt(m)
  editDialog.value = true
}

const finalPromptGen = ref(false)
async function generateFinalPrompt(m) {
  const epId = firstEpisodeId.value
  if (!epId) { toast.error(t('detail.mat.needEpisodeForPrompt')); return }
  finalPromptGen.value = true
  try {
    let res
    if (m.kindKey === 'character') res = await characterAPI.generatePrompt(m.id, epId, true)
    else if (m.kindKey === 'scene') res = await sceneAPI.generatePrompt(m.id, epId, true)
    else res = await propAPI.generatePrompt(m.id, epId, true)
    const fp = res?.final_prompt || res?.finalPrompt
    if (!fp) throw new Error(t('episode.asset.promptGenFailedRetry'))
    editTarget.value = { ...m, finalPrompt: fp, final_prompt: fp }
    editDraft.finalPrompt = fp
    toast.success(t('episode.asset.promptGenerated'))
  } catch (e) {
    toastError(e)
  } finally {
    finalPromptGen.value = false
  }
}

function closeEdit() {
  editDialog.value = false
  editTarget.value = null
}

async function saveEdit() {
  const target = editTarget.value
  if (!target) return
  if (target.kindKey === 'character' && !String(editDraft.name ?? '').trim()) { toast.error(t('episode.create.nameRequired')); return }
  if (target.kindKey === 'scene' && !String(editDraft.location ?? '').trim()) { toast.error(t('detail.mat.locationRequired')); return }
  if (target.kindKey === 'prop' && !String(editDraft.name ?? '').trim()) { toast.error(t('episode.create.nameRequired')); return }
  editSaving.value = true
  try {
    const fp = editDraft.finalPrompt || null
    if (target.kindKey === 'character') await characterAPI.update(target.id, { name: editDraft.name, role: editDraft.role, appearance: editDraft.appearance, description: editDraft.description, styling: editDraft.styling, finalPrompt: fp })
    else if (target.kindKey === 'scene') await sceneAPI.update(target.id, { location: editDraft.location, time: editDraft.time, prompt: editDraft.prompt, lighting: editDraft.lighting, finalPrompt: fp })
    else await propAPI.update(target.id, { name: editDraft.name, type: editDraft.type, description: editDraft.description, finalPrompt: fp })
    toast.success(t('common.saved'))
    closeEdit()
    emit('refresh')
  } catch (e) {
    toastError(e)
  } finally {
    editSaving.value = false
  }
}
</script>

<style scoped>
.assets-wrap {
  display: flex;
  flex-direction: column;
  gap: var(--sp-4);
}

.asset-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-3);
  flex-wrap: wrap;
}

.asset-filter .seg-item {
  display: inline-flex;
  align-items: center;
  gap: var(--sp-2);
}

.seg-count {
  padding: 1px 6px;
  border-radius: var(--radius-pill);
  background: var(--overlay-track);
  color: var(--text-3);
  font-size: 10.5px;
  font-weight: 600;
  line-height: 1.4;
}

.seg-item.on .seg-count {
  background: var(--accent-bg);
  color: var(--accent-text);
}

.asset-groups {
  display: flex;
  flex-direction: column;
  gap: var(--sp-6);
}

.asset-section {
  display: flex;
  flex-direction: column;
  gap: var(--sp-3);
}

.asset-group-head {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  padding: var(--sp-2) var(--sp-3);
  border-radius: var(--radius);
  border: 1px solid var(--border);
  border-left: 3px solid var(--text-3);
  background: var(--surface-raised);
  font-size: 13px;
  font-weight: 700;
  color: var(--text-1);
}

.asset-group-head .group-icon {
  display: inline-flex;
  color: var(--text-2);
}

.asset-group-head .group-label {
  letter-spacing: 0.01em;
}

.asset-group-head .group-count {
  margin-left: auto;
  font-size: 11px;
  font-weight: 600;
  color: var(--text-2);
  background: var(--overlay-track);
  border-radius: var(--radius-pill);
  padding: 1px 8px;
}

.asset-group-head.is-character {
  border-left-color: var(--accent);
  background: var(--accent-bg);
  border-color: transparent;
  border-left: 3px solid var(--accent);
  color: var(--accent-text);
}
.asset-group-head.is-character .group-icon { color: var(--accent-text); }

.asset-group-head.is-scene {
  border-left-color: var(--success);
  background: var(--success-bg);
  border-color: transparent;
  border-left: 3px solid var(--success);
  color: var(--tag-success-text);
}
.asset-group-head.is-scene .group-icon { color: var(--tag-success-text); }

.asset-group-head.is-prop {
  border-left-color: var(--warning);
  background: var(--warning-bg);
  border-color: transparent;
  border-left: 3px solid var(--warning);
  color: var(--tag-warning-text);
}
.asset-group-head.is-prop .group-icon { color: var(--tag-warning-text); }

.character-asset-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: var(--sp-4);
  align-items: stretch;
}

.asset-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(230px, 1fr));
  gap: var(--sp-4);
  align-items: stretch;
}

.asset-card,
.character-asset-card {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  border-radius: var(--radius-lg);
  cursor: pointer;
  transition: transform 0.18s var(--ease-out), box-shadow 0.18s var(--ease-out), border-color 0.18s var(--ease-out);
}

.asset-card:hover,
.character-asset-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-lift);
  border-color: var(--border-strong);
}

.asset-click-card:focus-visible,
.character-asset-card:focus-visible {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 3px var(--button-focus), var(--shadow-panel);
}

.asset-cover {
  position: relative;
  aspect-ratio: 1;
  background: var(--bg-2);
  overflow: hidden;
  border-bottom: 1px solid var(--border);
}

.asset-cover.wide {
  aspect-ratio: 16 / 9;
}

.asset-cover img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.previewable-image {
  cursor: zoom-in;
  transition: transform 0.2s var(--ease-out), filter 0.2s var(--ease-out);
}

.previewable-image:hover {
  transform: scale(1.02);
  filter: saturate(1.04);
}

.asset-cover-badge {
  position: absolute;
  top: var(--sp-2);
  left: var(--sp-2);
  display: inline-flex;
  align-items: center;
  padding: 2px var(--sp-2);
  border-radius: var(--radius-pill);
  background: var(--header-bg);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border: 1px solid var(--border);
  box-shadow: var(--shadow-xs);
  color: var(--text-2);
  font-size: 10px;
  font-weight: 700;
}

.asset-cover-badge.is-ready {
  background: var(--success-bg);
  border-color: transparent;
  color: var(--tag-success-text);
}

.asset-cover-badge.is-pending {
  background: var(--accent-bg);
  border-color: transparent;
  color: var(--accent-text);
}

.asset-cover-empty {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--text-3);
}

.character-body,
.asset-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: var(--sp-2);
  padding: var(--sp-3) var(--sp-4);
  min-width: 0;
}

.character-name-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-2);
  min-width: 0;
}

.character-name {
  font-size: 13.5px;
  font-weight: 600;
  line-height: 1.3;
  color: var(--text-0);
}

.character-visual-summary {
  display: flex;
  flex-direction: column;
  gap: 2px;
  color: var(--text-2);
  font-size: 11.5px;
  line-height: 1.45;
  min-width: 0;
}

.asset-final-prompt {
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding-top: var(--sp-2);
  border-top: 1px solid var(--border);
  font-size: 11px;
  line-height: 1.5;
}

.afp-label {
  font-size: 10px;
  font-weight: 700;
  letter-spacing: 0.04em;
  color: var(--text-3);
}

.afp-text {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  overflow: hidden;
  word-break: break-word;
  color: var(--text-2);
}

.afp-text.dim {
  color: var(--text-3);
}

.asset-name {
  font-size: 13.5px;
  font-weight: 600;
  color: var(--text-0);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.asset-meta {
  font-size: 11.5px;
  line-height: 1.5;
}

.asset-desc {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  overflow: hidden;
  word-break: break-word;
}

.asset-light {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.asset-final {
  margin-top: auto;
  padding-top: var(--sp-2);
  border-top: 1px solid var(--border);
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 2;
  overflow: hidden;
  word-break: break-word;
  color: var(--text-2);
}

.asset-final .afp-label {
  margin-right: var(--sp-1);
}

.prop-name-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-2);
}

.prop-name-row .asset-name {
  min-width: 0;
}

.asset-foot {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  padding: var(--sp-2) var(--sp-4);
  background: var(--surface-soft);
  border-top: 1px solid var(--border);
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--bg-3);
  flex-shrink: 0;
}

.dot.ok {
  background: var(--success);
}

.dot.pending {
  background: var(--accent);
}

.empty-state {
  min-height: 280px;
  padding: var(--sp-8) var(--sp-6);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: var(--sp-2);
  border: 1px dashed var(--border-strong);
  border-radius: var(--radius-lg);
  background: var(--surface-raised);
  text-align: center;
}

.empty-icon {
  width: 56px;
  height: 56px;
  border-radius: var(--radius-lg);
  background: var(--bg-2);
  color: var(--text-3);
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: var(--sp-1);
}

.empty-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-1);
}

.empty-desc {
  font-size: 12.5px;
  color: var(--text-3);
  max-width: 300px;
  line-height: 1.6;
}

/* Lightbox viewer */
.viewer-overlay {
  z-index: 130;
  padding: var(--sp-6);
}

.viewer-dialog {
  width: min(960px, calc(100vw - 48px));
  padding: var(--sp-4);
}

.viewer-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-3);
  margin-bottom: var(--sp-3);
}

.viewer-title {
  font-size: 13.5px;
  font-weight: 600;
  color: var(--text-1);
}

.viewer-stage {
  border-radius: var(--radius);
  background: var(--media-surface);
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.viewer-img {
  width: 100%;
  max-height: 76vh;
  object-fit: contain;
  display: block;
}

/* Detail / Edit dialog */
.mat-detail-overlay {
  z-index: 118;
  padding: var(--sp-6);
}

.mat-detail-dialog {
  width: min(1040px, calc(100vw - 48px));
  max-height: calc(100vh - 48px);
}

.mat-detail-head {
  justify-content: space-between;
  padding: var(--sp-4) var(--sp-6);
}

.mat-detail-title-block {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: var(--sp-1);
}

.mat-detail-title {
  margin: 0;
  color: var(--text-0);
  font-size: 18px;
  line-height: 1.2;
  font-family: var(--font-display);
}

.mat-detail-head-actions {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  flex-shrink: 0;
}

.mat-detail-body {
  min-height: 0;
  overflow: auto;
  padding: var(--sp-5) var(--sp-6);
}

.mat-detail-shell {
  display: grid;
  grid-template-columns: minmax(280px, 360px) minmax(0, 1fr);
  gap: var(--sp-5);
  align-items: start;
}

.mat-detail-preview-panel,
.mat-detail-editor-panel {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: var(--sp-3);
}

.mat-detail-preview-panel {
  position: sticky;
  top: 0;
}

.mat-detail-section-title {
  min-height: 24px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-3);
  color: var(--text-1);
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0.02em;
}

.mat-detail-section-title .dim {
  font-size: 11px;
  font-weight: 500;
  letter-spacing: 0;
  text-align: right;
}

.mat-detail-prompt-panel {
  margin-top: var(--sp-5);
  border-top: 1px solid var(--border);
  padding-top: var(--sp-4);
}

.mat-detail-prompt-gen {
  margin-left: auto;
  flex-shrink: 0;
}

.mat-detail-prompt-text {
  width: 100%;
  margin-top: var(--sp-3);
  background: var(--surface-input);
  font-size: 12.5px;
  line-height: 1.65;
  resize: vertical;
  min-height: 112px;
}

.mat-detail-state {
  min-height: 20px;
  display: inline-flex;
  align-items: center;
  padding: 0 var(--sp-2);
  border-radius: var(--radius-pill);
  background: var(--overlay-track);
  color: var(--text-3);
  font-size: 10.5px;
  font-weight: 700;
  white-space: nowrap;
}

.mat-detail-state.is-ready {
  color: var(--tag-success-text);
  background: var(--success-bg);
}

.mat-detail-media-frame {
  position: relative;
  width: 100%;
  aspect-ratio: 16 / 9;
  display: block;
  padding: 0;
  border: 1px solid var(--surface-outline);
  border-radius: var(--radius);
  background: var(--bg-2);
  color: var(--text-3);
  overflow: hidden;
  cursor: zoom-in;
}

.mat-detail-media-frame:disabled {
  cursor: default;
  opacity: 1;
}

.mat-detail-media-frame:focus-visible {
  outline: none;
  border-color: var(--border-focus);
  box-shadow: 0 0 0 3px var(--button-focus);
}

.mat-detail-media-frame img {
  width: 100%;
  height: 100%;
  display: block;
  object-fit: cover;
}

.mat-detail-media-empty {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--text-3);
}

.mat-detail-meta-row {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--sp-2);
}

.mat-detail-meta-item {
  min-width: 0;
  padding: var(--sp-2) var(--sp-3);
  border: 1px solid var(--surface-outline);
  border-radius: var(--radius);
  background: var(--surface-muted);
}

.mat-detail-meta-item span {
  display: block;
  color: var(--text-3);
  font-size: 10.5px;
  font-weight: 600;
  letter-spacing: 0.03em;
}

.mat-detail-meta-item strong {
  display: block;
  margin-top: var(--sp-1);
  min-width: 0;
  color: var(--text-0);
  font-size: 12.5px;
  line-height: 1.35;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.mat-detail-edit-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--sp-3);
}

.mat-detail-edit-grid--character,
.mat-detail-edit-grid--scene {
  grid-template-columns: 1fr;
}

.mat-detail-edit-pair {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: var(--sp-3);
}

.mat-detail-edit-field {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.mat-detail-edit-field--full {
  grid-column: 1 / -1;
}

.mat-detail-edit-field > span {
  color: var(--text-2);
  font-size: 11.5px;
  font-weight: 600;
}

.mat-detail-textarea {
  min-height: 116px;
  resize: vertical;
}

.mat-detail-foot {
  justify-content: space-between;
  padding: var(--sp-3) var(--sp-6);
}

.mat-detail-secondary-actions,
.mat-detail-primary-actions {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  flex-wrap: wrap;
}

@media (max-width: 860px) {
  .mat-detail-overlay {
    padding: var(--sp-4);
  }
  .mat-detail-dialog {
    width: calc(100vw - 32px);
    max-height: calc(100vh - 32px);
  }
  .mat-detail-shell {
    grid-template-columns: 1fr;
  }
  .mat-detail-preview-panel {
    position: static;
  }
  .mat-detail-edit-grid,
  .mat-detail-edit-pair {
    grid-template-columns: 1fr;
  }
  .mat-detail-foot {
    flex-direction: column-reverse;
    align-items: stretch;
  }
  .mat-detail-primary-actions {
    justify-content: flex-end;
  }
}
</style>
