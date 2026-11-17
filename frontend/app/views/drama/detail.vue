<template>
  <div class="page" v-if="drama">
    <!-- 剧集策划台 Hero 横幅 -->
    <header class="page-head card">
      <div class="head-glow" aria-hidden="true" />
      <button type="button" class="back-btn" :title="t('common.back')" :aria-label="t('common.back')" @click="navigateTo('/')">
        <ArrowLeft :size="16" :stroke-width="2" />
      </button>

      <div class="head-info">
        <div class="head-title-row">
          <h1 class="page-title">{{ drama.title }}</h1>
          <span v-if="dramaStyleLabel" class="tag tag-accent">{{ dramaStyleLabel }}</span>
          <span v-if="drama.aspect_ratio && drama.aspect_ratio !== 'adaptive'" class="tag mono">{{ drama.aspect_ratio }}</span>
        </div>
        <div class="page-meta">
          <span class="meta-item">
            <Users :size="13" :stroke-width="1.9" />
            {{ t('detail.head.charCount', { n: drama.characters?.length || 0 }) }}
          </span>
          <span class="meta-item">
            <MapPin :size="13" :stroke-width="1.9" />
            {{ t('detail.head.sceneCount', { n: drama.scenes?.length || 0 }) }}
          </span>
          <span class="meta-item">
            <Film :size="13" :stroke-width="1.9" />
            {{ t('detail.head.epCount', { n: drama.episodes?.length || 0 }) }}
          </span>
        </div>
      </div>

      <div v-if="heroThumbs.length" class="head-preview-strip" aria-hidden="true">
        <div v-for="(img, idx) in heroThumbs" :key="idx" class="head-preview-thumb">
          <img :src="img.thumb" alt="" @error="thumbFallback($event, img.orig)" />
        </div>
      </div>

      <div class="head-actions">
        <button v-if="latestEpisode" type="button" class="btn" @click="openEpisode(latestEpisode)">
          <Play :size="13" :stroke-width="2.1" />
          <span>{{ t('detail.ep.enter') }}</span>
        </button>
        <button type="button" class="btn btn-primary head-action" @click="openAddEpisode">
          <Plus :size="14" :stroke-width="2.2" />
          {{ t('detail.head.addEpisode') }}
        </button>
      </div>
    </header>

    <!-- 主 Tab：剧集列表 / 素材库 -->
    <nav class="page-tabs" role="tablist">
      <button type="button" role="tab" :aria-selected="activeTab === 'episodes'" :class="['tab-btn', { on: activeTab === 'episodes' }]" @click="activeTab = 'episodes'">
        <Film :size="15" :stroke-width="1.9" />
        <span>{{ t('detail.tabs.episodes') }}</span>
        <span class="tab-count mono">{{ drama.episodes?.length || 0 }}</span>
      </button>
      <button type="button" role="tab" :aria-selected="activeTab === 'assets'" :class="['tab-btn', { on: activeTab === 'assets' }]" @click="activeTab = 'assets'">
        <Images :size="15" :stroke-width="1.9" />
        <span>{{ t('detail.tabs.assets') }}</span>
        <span v-if="assetTotal > 0" class="tab-count mono">{{ assetTotal }}</span>
      </button>
    </nav>

    <!-- 剧集列表 -->
    <div v-show="activeTab === 'episodes'" class="ep-grid">
      <article
        v-for="(ep, i) in drama.episodes"
        :key="ep.id"
        :class="['card', 'ep-card', `ep-card-${epStatus(ep)}`]"
        :style="{ animationDelay: `${i * 0.04}s` }"
        tabindex="0"
        role="button"
        @click="openEpisode(ep)"
        @keydown.enter.self.prevent="openEpisode(ep)"
        @keydown.space.self.prevent="openEpisode(ep)"
      >
        <div class="ep-card-top">
          <div v-if="epPoster(ep)" class="ep-poster-thumb">
            <img :src="epPoster(ep)" :alt="ep.title" loading="lazy" />
            <span class="ep-poster-badge mono">EP{{ String(ep.episode_number || ep.episodeNumber).padStart(2, '0') }}</span>
          </div>
          <div v-else :class="['ep-number', `ep-num-${epStatus(ep)}`]">
            <span class="ep-num-label">EP</span>
            <b>{{ String(ep.episode_number || ep.episodeNumber).padStart(2, '0') }}</b>
          </div>

          <div class="ep-main">
            <h3 class="ep-title">{{ ep.title }}</h3>
            <div class="ep-meta-row">
              <span v-if="ep.duration" class="ep-meta">
                <Clock :size="12" :stroke-width="1.9" />
                {{ ep.duration }}s
              </span>
              <span v-if="ep.scriptContent || ep.script_content" class="ep-meta ep-meta-ok">
                <FileText :size="12" :stroke-width="1.9" />
                {{ t('detail.ep.scriptReady') }}
              </span>
              <span v-if="ep.videoUrl || ep.video_url" class="ep-meta ep-meta-ok">
                <Video :size="12" :stroke-width="1.9" />
                {{ t('detail.ep.merged') }}
              </span>
              <span v-if="ep.updatedAt || ep.updated_at" class="ep-meta ep-time">
                {{ formatEpTime(ep.updatedAt || ep.updated_at) }}
              </span>
            </div>
          </div>

          <div class="ep-badges" @click.stop>
            <AppMenu
              :open="epStatusMenuId === ep.id"
              placement="bottom-end"
              :min-width="116"
              @update:open="(v) => { epStatusMenuId = v ? ep.id : null }"
            >
              <template #trigger>
                <button type="button" :class="['tag', 'ep-status-btn', `ep-status-${epStatus(ep)}`]" :title="t('detail.ep.statusTitle')">
                  <span :class="['status-dot', epStatusDotClass(ep)]"></span>
                  {{ epStatusLabel(ep) }}
                </button>
              </template>
              <AppMenuItem
                v-for="s in epStatusOptions"
                :key="s.value"
                :selected="epStatus(ep) === s.value"
                @click="setEpisodeStatus(ep, s.value)"
              >{{ s.label }}</AppMenuItem>
            </AppMenu>
          </div>
        </div>

        <div class="ep-card-foot" @click.stop>
          <AppMenu
            :open="epResMenuId === ep.id"
            placement="top-start"
            :min-width="116"
            @update:open="(v) => { epResMenuId = v ? ep.id : null }"
          >
            <template #trigger>
              <button type="button" class="tag ep-res-btn mono" :title="t('detail.ep.resTitle')">
                {{ epResolution(ep) }}
              </button>
            </template>
            <AppMenuItem
              v-for="r in resolutionOptions"
              :key="r.value"
              :selected="epResolution(ep) === r.value"
              @click="setEpisodeResolution(ep, r.value)"
            >{{ r.label }}</AppMenuItem>
          </AppMenu>

          <span class="ep-foot-spacer"></span>

          <button
            class="btn btn-icon btn-sm ep-delete"
            type="button"
            :title="t('detail.ep.deleteTitle')"
            @click="episodeToDelete = ep"
          >
            <Trash2 :size="14" :stroke-width="1.8" />
          </button>

          <button type="button" class="ep-enter" @click="openEpisode(ep)">
            <span>{{ t('detail.ep.enter') }}</span>
            <ChevronRight :size="13" :stroke-width="2.1" />
          </button>
        </div>
      </article>

      <!-- Empty / Add Next Episode Card -->
      <div
        v-if="!drama.episodes?.length"
        class="card ep-empty"
        role="button"
        tabindex="0"
        :title="t('detail.ep.emptyCreateTitle')"
        @click="openAddEpisode"
        @keydown.enter="openAddEpisode"
        @keydown.space.prevent="openAddEpisode"
      >
        <div class="ep-empty-icon">
          <Plus :size="16" :stroke-width="2" />
        </div>
        <p>{{ t('detail.ep.emptyCreateText') }}</p>
      </div>

      <div
        v-else
        class="card ep-empty ep-add"
        role="button"
        tabindex="0"
        :title="t('detail.ep.addNextTitle', { n: (drama.episodes?.length || 0) + 1 })"
        @click="openAddEpisode"
        @keydown.enter="openAddEpisode"
        @keydown.space.prevent="openAddEpisode"
      >
        <div class="ep-empty-icon">
          <Plus :size="16" :stroke-width="2" />
        </div>
        <p>{{ t('detail.ep.addNextText', { n: (drama.episodes?.length || 0) + 1 }) }}</p>
      </div>
    </div>

    <!-- 素材库 -->
    <AssetGallery
      v-show="activeTab === 'assets'"
      :drama="drama"
      @refresh="load"
      @voice-updated="updateCharacterVoice"
    />

    <!-- 创建新集弹窗 -->
    <Teleport to="body">
      <div v-if="addDialog" class="overlay" @click.self="addDialog = false">
        <div class="dialog ep-dialog">
          <div class="dialog-head">
            <div class="modal-icon">
              <Clapperboard :size="18" :stroke-width="1.8" />
            </div>
            <div class="dialog-title">{{ t('detail.epCreate.title') }}</div>
            <button type="button" class="btn btn-icon btn-sm btn-ghost ml-auto dialog-close" :aria-label="t('common.close')" @click="addDialog = false">
              <X :size="15" />
            </button>
          </div>
          <div class="dialog-body ep-dialog-body">
            <label class="field">
              <span class="field-label">{{ t('detail.epCreate.titleField') }}</span>
              <input v-model="newEpisodeTitle" class="input" :placeholder="t('detail.epCreate.titlePlaceholder')" />
              <span class="field-hint">{{ t('detail.epCreate.titleHint', { example: t('detail.epCreate.example', { n: 3 }) }) }}</span>
            </label>
            <label class="field">
              <span class="field-label">{{ t('detail.epCreate.resolution') }}</span>
              <BaseSelect v-model="newEpisodeResolution" :options="resolutionOptions" :placeholder="t('detail.epCreate.resolutionPlaceholder')" />
              <span class="field-hint">{{ t('detail.epCreate.resolutionHint') }}</span>
            </label>
          </div>
          <div class="dialog-foot">
            <span class="dialog-foot-copy">{{ t('detail.epCreate.lockCopy') }}</span>
            <button type="button" class="btn" @click="addDialog = false">{{ t('common.cancel') }}</button>
            <button type="button" class="btn btn-primary" :disabled="creatingEpisode" @click="addEpisode">
              <Loader2 v-if="creatingEpisode" :size="13" class="animate-spin" />
              {{ creatingEpisode ? t('detail.epCreate.creating') : t('detail.epCreate.create') }}
            </button>
          </div>
        </div>
      </div>
    </Teleport>

    <ConfirmDialog
      :open="!!episodeToDelete"
      :title="t('detail.ep.deleteTitle')"
      :message="t('detail.epDelete.message', { title: episodeToDelete?.title || t('detail.ep.episodeN', { n: episodeToDelete?.episode_number || episodeToDelete?.episodeNumber }) })"
      :loading="deletingEpisode"
      @confirm="confirmDelEpisode"
      @cancel="episodeToDelete = null"
    />
  </div>
</template>

<script setup>
import { toast } from 'vue-sonner'
import { useI18n } from 'vue-i18n'
import {
  ArrowLeft,
  ChevronRight,
  Clapperboard,
  Clock,
  FileText,
  Film,
  Images,
  Loader2,
  MapPin,
  Play,
  Plus,
  Trash2,
  Users,
  Video,
  X,
} from 'lucide-vue-next'
import { toastError } from '~/composables/useToast'
import { dramaAPI, episodeAPI, stylePresetAPI } from '~/composables/useApi'
import { thumbOf, thumbFallback, posterOf } from '~/composables/useMedia'
import BaseSelect from '~/components/BaseSelect.vue'
import AssetGallery from '~/components/AssetGallery.vue'

const { t } = useI18n()

const route = useRoute()
const drama = ref(null)
const stylePresets = ref([])
const dramaId = Number(route.params.id)
const activeTab = ref('episodes')
const addDialog = ref(false)
const creatingEpisode = ref(false)
const newEpisodeTitle = ref('')
const episodeToDelete = ref(null)
const deletingEpisode = ref(false)

const dramaStyleLabel = computed(() => {
  const key = drama.value?.style
  if (!key) return ''
  return stylePresets.value.find(p => p.value === key)?.name || key
})

const assetTotal = computed(() => {
  const d = drama.value
  if (!d) return 0
  return (d.characters?.length || 0) + (d.scenes?.length || 0) + (d.props?.length || 0)
})

const latestEpisode = computed(() => {
  const eps = drama.value?.episodes || []
  if (!eps.length) return null
  return [...eps].sort((a, b) => Number(b.episode_number || b.episodeNumber || 0) - Number(a.episode_number || a.episodeNumber || 0))[0]
})

function normalizeMedia(raw) {
  if (!raw) return ''
  return /^https?:\/\//i.test(raw) || raw.startsWith('/') ? raw : `/${raw}`
}

function epPoster(ep) {
  const v = normalizeMedia(ep?.videoUrl || ep?.video_url || '')
  return v ? posterOf(v) : ''
}

const heroThumbs = computed(() => {
  const d = drama.value
  if (!d) return []
  const out = []
  const seen = new Set()
  const add = (raw) => {
    const orig = normalizeMedia(raw)
    if (!orig) return
    const thumb = thumbOf(orig)
    if (!thumb || seen.has(thumb)) return
    seen.add(thumb)
    out.push({ thumb, orig })
  }
  for (const s of d.scenes || []) {
    if (s.image_url || s.imageUrl) add(s.image_url || s.imageUrl)
    if (out.length >= 3) return out
  }
  for (const c of d.characters || []) {
    if (c.image_url || c.imageUrl) add(c.image_url || c.imageUrl)
    if (out.length >= 3) return out
  }
  return out
})

const resolutionOptions = computed(() => ([
  { label: t('detail.ep.res720'), value: '720p' },
  { label: t('detail.ep.res480'), value: '480p' },
]))
const newEpisodeResolution = ref('720p')
const epResMenuId = ref(null)

function epResolution(ep) { return ep.resolution === '480p' ? '480p' : '720p' }

async function setEpisodeResolution(ep, resolution) {
  epResMenuId.value = null
  if (epResolution(ep) === resolution) return
  const prev = ep.resolution
  ep.resolution = resolution
  try {
    await episodeAPI.update(ep.id, { resolution })
    toast.success(t('detail.ep.resSwitched', { res: resolution }))
  } catch (e) {
    ep.resolution = prev
    toastError(e)
  }
}

const epStatusOptions = computed(() => ([
  { label: t('index.status.draft'), value: 'draft' },
  { label: t('index.status.active'), value: 'active' },
  { label: t('index.status.completed'), value: 'completed' },
]))
const epStatusMenuId = ref(null)

function epStatus(ep) { return ep.status || 'draft' }
function epStatusLabel(ep) { return epStatusOptions.value.find(s => s.value === epStatus(ep))?.label || t('index.status.draft') }
function epStatusDotClass(ep) { return epStatus(ep) === 'active' ? 'dot-active' : epStatus(ep) === 'completed' ? 'dot-done' : 'dot-pending' }

function formatEpTime(ts) {
  if (!ts) return ''
  const d = new Date(ts)
  const now = new Date()
  const diff = now - d
  if (diff < 60_000) return t('index.time.justNow')
  if (diff < 3600_000) return t('index.time.minutesAgo', { n: Math.floor(diff / 60000) })
  if (diff < 86400_000) return t('index.time.hoursAgo', { n: Math.floor(diff / 3600000) })
  return `${d.getMonth() + 1}/${d.getDate()}`
}

async function setEpisodeStatus(ep, status) {
  epStatusMenuId.value = null
  if (epStatus(ep) === status) return
  const prev = ep.status
  ep.status = status
  try {
    await episodeAPI.update(ep.id, { status })
  } catch (e) {
    ep.status = prev
    toastError(e)
  }
}

function openEpisode(ep) {
  navigateTo(`/drama/${drama.value.id}/episode/${ep.episode_number || ep.episodeNumber}`)
}

async function load() {
  try {
    const [d, presets] = await Promise.all([
      dramaAPI.get(dramaId),
      stylePresets.value.length ? Promise.resolve(stylePresets.value) : stylePresetAPI.list().catch(() => []),
    ])
    drama.value = d
    if (Array.isArray(presets)) stylePresets.value = presets
  } catch (e) {
    toastError(e)
  }
}

function updateCharacterVoice(id, voice) {
  const patch = { ...voice, voiceAudioUrl: voice.voice_audio_url, voiceAudioName: voice.voice_audio_name }
  const character = drama.value?.characters?.find(item => item.id === id)
  if (character) Object.assign(character, patch)
}

function openAddEpisode() {
  newEpisodeTitle.value = ''
  newEpisodeResolution.value = '720p'
  addDialog.value = true
}

async function addEpisode() {
  try {
    creatingEpisode.value = true
    await episodeAPI.create({
      drama_id: dramaId,
      title: newEpisodeTitle.value || undefined,
      resolution: newEpisodeResolution.value,
    })
    toast.success(t('detail.epCreate.added'))
    addDialog.value = false
    load()
  } catch (e) {
    toastError(e)
  } finally {
    creatingEpisode.value = false
  }
}

async function confirmDelEpisode() {
  const ep = episodeToDelete.value
  if (!ep) return
  try {
    deletingEpisode.value = true
    await episodeAPI.del(ep.id)
    toast.success(t('index.deleted'))
    episodeToDelete.value = null
    load()
  } catch (e) {
    toastError(e)
  } finally {
    deletingEpisode.value = false
  }
}

onMounted(load)
</script>

<style scoped>
.page {
  padding: var(--sp-5) var(--sp-8) var(--sp-10);
  overflow-y: auto;
  height: 100%;
  animation: fadeUp 0.35s var(--ease-out) both;
}

/* Header card */
.page-head {
  position: relative;
  display: flex;
  align-items: center;
  gap: var(--sp-4);
  padding: 18px 22px;
  border-radius: var(--radius-xl);
  margin-bottom: var(--sp-4);
  overflow: hidden;
}
.head-glow {
  position: absolute;
  inset: 0;
  background: var(--hero-ambient);
  pointer-events: none;
}
.page-head > *:not(.head-glow) {
  position: relative;
  z-index: 1;
}
.head-preview-strip {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-shrink: 0;
}
.head-preview-thumb {
  width: 56px;
  height: 40px;
  border-radius: var(--radius-sm);
  overflow: hidden;
  border: 1px solid var(--border);
  background: var(--media-surface);
}
.head-preview-thumb img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}
.head-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}
.ep-poster-thumb {
  position: relative;
  width: 72px;
  height: 44px;
  border-radius: var(--radius-sm);
  overflow: hidden;
  flex-shrink: 0;
  background: var(--media-surface);
  border: 1px solid var(--border);
}
.ep-poster-thumb img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}
.ep-poster-badge {
  position: absolute;
  left: 4px;
  bottom: 4px;
  padding: 1px 5px;
  border-radius: 4px;
  background: rgba(6, 7, 11, 0.78);
  color: #fff;
  font-size: 9px;
  font-weight: 700;
}

.back-btn {
  width: var(--button-height);
  height: var(--button-height);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  background: var(--surface-soft);
  color: var(--text-1);
  cursor: pointer;
  transition: background 0.16s var(--ease-out), color 0.16s var(--ease-out), border-color 0.16s var(--ease-out), box-shadow 0.16s var(--ease-out);
}
.back-btn:hover {
  background: var(--bg-hover);
  border-color: var(--border-strong);
  color: var(--text-0);
}
.back-btn:focus-visible {
  outline: none;
  box-shadow: 0 0 0 3.5px var(--button-focus);
}

.head-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: var(--sp-1);
}

.head-title-row {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  flex-wrap: wrap;
  min-width: 0;
}

.page-title {
  font-size: 18px;
  font-weight: 700;
  letter-spacing: -0.02em;
  line-height: 1.25;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.page-meta {
  display: flex;
  align-items: center;
  gap: var(--sp-4);
  flex-wrap: wrap;
}

.meta-item {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  font-size: 12px;
  color: var(--text-2);
}

.head-action {
  flex-shrink: 0;
}

/* 主 Tab 导航 */
.page-tabs {
  display: flex;
  align-items: center;
  gap: var(--sp-1);
  border-bottom: 1px solid var(--border);
  margin-bottom: var(--sp-4);
}

.tab-btn {
  position: relative;
  display: inline-flex;
  align-items: center;
  gap: var(--sp-2);
  border: none;
  background: transparent;
  padding: var(--sp-2) var(--sp-1) 10px;
  margin-right: var(--sp-5);
  font-size: 13.5px;
  font-weight: 600;
  color: var(--text-2);
  cursor: pointer;
  transition: color 0.16s var(--ease-out);
}
.tab-btn svg { opacity: 0.75; }
.tab-btn::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  bottom: -1px;
  height: 2px;
  border-radius: var(--radius-pill);
  background: transparent;
  transition: background 0.16s var(--ease-out);
}
.tab-btn:hover { color: var(--text-0); }
.tab-btn.on { color: var(--text-0); }
.tab-btn.on::after { background: var(--accent); }

.tab-count {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 20px;
  height: 18px;
  padding: 0 6px;
  border-radius: var(--radius-pill);
  font-size: 11px;
  font-weight: 700;
  background: var(--overlay-track);
  color: var(--text-2);
}
.tab-btn.on .tab-count {
  background: var(--accent-bg);
  color: var(--accent-text);
}

/* Episode Grid */
.ep-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(340px, 1fr));
  gap: var(--sp-3);
}

.ep-card {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: var(--sp-3);
  padding: var(--sp-4);
  border-radius: var(--radius-lg);
  cursor: pointer;
  animation: fadeUp 0.3s var(--ease-out) both;
  transition: border-color 0.18s var(--ease-out), box-shadow 0.18s var(--ease-out), transform 0.18s var(--ease-out);
}
.ep-card:hover {
  border-color: var(--border-strong);
  box-shadow: var(--shadow-lift);
  transform: translateY(-1px);
}
.ep-card:focus-visible {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 3px var(--button-focus);
}

.ep-card-top {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  min-width: 0;
}

.ep-number {
  width: 44px;
  height: 44px;
  flex-shrink: 0;
  border-radius: var(--radius);
  border: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-family: var(--font-mono);
  background: var(--surface-soft);
  color: var(--text-1);
}
.ep-num-active {
  background: var(--accent-bg);
  border-color: transparent;
  color: var(--accent-text);
}
.ep-num-label {
  font-size: 8px;
  letter-spacing: 0.16em;
  font-weight: 700;
  opacity: 0.65;
  line-height: 1;
  margin-bottom: 2px;
}
.ep-number b {
  font-size: 15px;
  font-weight: 700;
  line-height: 1;
}

.ep-main {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: var(--sp-1);
}

.ep-title {
  font-size: 14px;
  font-weight: 600;
  color: var(--text-0);
  line-height: 1.35;
  margin: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.ep-meta-row {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  flex-wrap: nowrap;
  overflow: hidden;
  white-space: nowrap;
}

.ep-meta {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11.5px;
  color: var(--text-3);
  white-space: nowrap;
  flex-shrink: 0;
}
.ep-meta svg { opacity: 0.75; flex-shrink: 0; }
.ep-meta-ok { color: var(--text-2); }
.ep-time { flex-shrink: 1; overflow: hidden; text-overflow: ellipsis; }

.ep-badges {
  display: flex;
  align-items: center;
  flex-shrink: 0;
}

.ep-card-foot {
  display: flex;
  align-items: center;
  gap: var(--sp-2);
  padding-top: var(--sp-3);
  border-top: 1px solid var(--border);
}
.ep-foot-spacer { flex: 1; }

.ep-enter {
  display: inline-flex;
  align-items: center;
  gap: var(--sp-1);
  padding: 5px var(--sp-3);
  border: none;
  border-radius: var(--radius-pill);
  background: var(--accent-bg);
  color: var(--accent-text);
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.16s var(--ease-out), color 0.16s var(--ease-out), box-shadow 0.16s var(--ease-out);
  white-space: nowrap;
}
.ep-enter:hover {
  background: var(--accent-gradient);
  color: var(--on-accent);
  box-shadow: 0 2px 8px var(--accent-glow);
}
.ep-enter svg { transition: transform 0.18s var(--ease-out); }
.ep-enter:hover svg { transform: translateX(2px); }

.ep-status-btn {
  cursor: pointer;
  border: 1px solid var(--border);
  font: inherit;
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 3px 10px;
  border-radius: var(--radius-pill);
  background: var(--surface-soft);
  color: var(--text-2);
  font-size: 11px;
  font-weight: 600;
  transition: background 0.14s var(--ease-out), color 0.14s var(--ease-out);
  white-space: nowrap;
}
.ep-status-btn:hover {
  color: var(--text-0);
  background: var(--bg-hover);
}

.ep-res-btn {
  cursor: pointer;
  border: 1px solid var(--border);
  font-size: 11px;
  font-weight: 600;
  display: inline-flex;
  align-items: center;
  padding: 3px var(--sp-2);
  border-radius: var(--radius-sm);
  background: var(--surface-soft);
  color: var(--text-2);
  transition: background 0.14s var(--ease-out), color 0.14s var(--ease-out);
}
.ep-res-btn:hover {
  background: var(--bg-hover);
  color: var(--text-0);
}

.status-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  flex-shrink: 0;
}
.dot-active { background: var(--success); }
.dot-done { background: var(--accent); }
.dot-pending { background: var(--text-3); }

.ep-delete {
  color: var(--text-3);
  background: transparent;
  box-shadow: none;
  transition: color 0.15s var(--ease-out), background 0.15s var(--ease-out);
}
.ep-delete:hover {
  color: var(--action-danger);
  background: var(--action-danger-bg);
}

/* Empty / Add card */
.ep-empty {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;
  gap: var(--sp-2);
  min-height: 116px;
  padding: var(--sp-4);
  text-align: center;
  color: var(--text-2);
  font-size: 13px;
  font-weight: 600;
  border: 1px dashed var(--border-strong);
  border-radius: var(--radius-lg);
  background: var(--surface-soft);
  box-shadow: none;
  cursor: pointer;
  transition: border-color 0.2s var(--ease-out), color 0.2s var(--ease-out), background 0.2s var(--ease-out);
}
.ep-empty:hover {
  border-color: var(--accent);
  color: var(--accent-text);
  background: var(--accent-bg);
}
.ep-empty-icon {
  width: 30px;
  height: 30px;
  border-radius: var(--radius-pill);
  background: var(--overlay-track);
  color: var(--text-2);
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: background 0.2s var(--ease-out), color 0.2s var(--ease-out);
}
.ep-empty:hover .ep-empty-icon {
  background: var(--surface-raised);
  color: var(--accent-text);
}

/* Create Episode Dialog */
.ep-dialog {
  width: min(480px, calc(100vw - 32px));
}
.modal-icon {
  width: 38px;
  height: 38px;
  flex: 0 0 auto;
  border-radius: var(--radius);
  background: var(--accent-bg);
  color: var(--accent-text);
  display: flex;
  align-items: center;
  justify-content: center;
}
.dialog-close {
  flex-shrink: 0;
  color: var(--text-2);
}
.ep-dialog-body {
  display: flex;
  flex-direction: column;
  gap: var(--sp-4);
}
.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.field-label {
  font-size: 12px;
  font-weight: 600;
  color: var(--text-1);
}
.field-hint {
  font-size: 11.5px;
  color: var(--text-3);
  line-height: 1.5;
}
.dialog-foot-copy {
  margin-right: auto;
  font-size: 11.5px;
  line-height: 1.5;
  color: var(--text-3);
}

@media (max-width: 860px) {
  .page {
    padding: var(--sp-4) var(--sp-4) var(--sp-8);
  }
  .page-head {
    flex-wrap: wrap;
  }
  .ep-grid {
    grid-template-columns: 1fr;
  }
  .dialog-foot {
    flex-wrap: wrap;
    gap: var(--sp-2);
  }
  .dialog-foot-copy {
    display: none;
  }
}
</style>
