<template>
  <div class="page">
    <!-- 造梦工厂 Hero 启动区：对标主流 AI 视频创作工作台 -->
    <section class="launcher-hero">
      <div class="hero-glow" aria-hidden="true" />
      <div class="hero-main">
        <div class="hero-badge">
          <Sparkles :size="12" :stroke-width="2.2" />
          <span>{{ t('app.title') }} · DREAMFACTORY STUDIO</span>
        </div>
        <h1 class="launcher-title">{{ t('index.hero.title') }}</h1>
        <p class="launcher-sub">{{ t('index.hero.sub') }}</p>
        <div class="hero-pipeline" aria-hidden="true">
          <span class="pipe-pill"><FileText :size="12" /> {{ t('episode.sidebar.scriptSection') }}</span>
          <ChevronRight :size="12" class="pipe-arrow" />
          <span class="pipe-pill"><Users :size="12" /> {{ t('episode.prod.assets') }}</span>
          <ChevronRight :size="12" class="pipe-arrow" />
          <span class="pipe-pill"><Clapperboard :size="12" /> {{ t('episode.prod.videos') }}</span>
          <ChevronRight :size="12" class="pipe-arrow" />
          <span class="pipe-pill"><Film :size="12" /> {{ t('episode.sidebar.exportSection') }}</span>
        </div>
      </div>

      <div class="hero-side">
        <div class="hero-metrics">
          <div class="metric-tile">
            <span class="metric-value mono">{{ dramas.length }}</span>
            <span class="metric-label">{{ t('index.hero.projectCount', { n: dramas.length }) }}</span>
          </div>
          <div class="metric-tile is-active">
            <span class="metric-value mono">{{ activeCount }}</span>
            <span class="metric-label">{{ t('index.hero.activeCount', { n: activeCount }) }}</span>
          </div>
          <div class="metric-tile is-accent">
            <span class="metric-value mono">{{ stylePresets.length }}</span>
            <span class="metric-label">{{ t('index.hero.styleCount', { n: stylePresets.length }) }}</span>
          </div>
        </div>
        <div class="head-actions">
          <button type="button" class="btn btn-icon tour-help-btn" :title="t('tour.helpTitle')" @click="startTour('index', INDEX_TOUR, t)">
            <CircleHelp :size="15" :stroke-width="1.8" />
          </button>
          <button type="button" class="btn btn-primary hero-create-btn" @click="showCreate = true">
            <Plus :size="15" :stroke-width="2.3" />
            <span>{{ t('index.create') }}</span>
          </button>
        </div>
      </div>
    </section>

    <!-- 工具栏：搜索 + 状态分段筛选 + 排序 -->
    <div class="toolbar">
      <label class="search-box">
        <Search :size="15" :stroke-width="1.8" class="search-icon" />
        <input v-model.trim="searchKeyword" class="input" :placeholder="t('index.searchPlaceholder')" />
      </label>

      <div class="seg filter-seg" role="tablist" :aria-label="t('index.status.all')">
        <button
          v-for="f in filters"
          :key="f.value"
          type="button"
          role="tab"
          :aria-selected="statusFilter === f.value"
          class="seg-item"
          :class="{ on: statusFilter === f.value }"
          @click="statusFilter = f.value"
        >
          {{ f.label }}
        </button>
      </div>

      <div class="sort-select-wrap">
        <BaseSelect v-model="sortMode" :options="sortOptions" :searchable="false" />
      </div>
    </div>

    <!-- 加载骨架：卡片 -->
    <div v-if="loading" class="project-grid">
      <div v-for="i in 8" :key="i" class="card skeleton-card">
        <div class="skeleton-cover"></div>
        <div class="skeleton-body">
          <div class="skeleton-line w-60"></div>
          <div class="skeleton-line w-40"></div>
        </div>
      </div>
    </div>

    <!-- 项目海报卡片网格 -->
    <div v-else-if="filteredDramas.length" class="project-grid">
      <article
        v-for="(d, i) in filteredDramas"
        :key="d.id"
        class="card project-card"
        :style="{ animationDelay: `${i * 0.04}s` }"
        tabindex="0"
        role="button"
        :aria-label="t('index.openProjectAria', { title: d.title })"
        @click="openDrama(d)"
        @keydown.enter.prevent="openDrama(d)"
        @keydown.space.prevent="openDrama(d)"
      >
        <div class="project-cover" :class="{ 'has-media': projectCovers(d).length > 0 }">
          <!-- 有已生成资产/成片海报帧时展示画报拼贴 -->
          <div
            v-if="projectCovers(d).length"
            :class="['cover-collage', `collage-${Math.min(projectCovers(d).length, 3)}`]"
          >
            <div v-for="(item, idx) in projectCovers(d)" :key="idx" class="collage-cell">
              <img
                :src="item.thumb"
                :alt="d.title"
                loading="lazy"
                @error="thumbFallback($event, item.orig)"
              />
            </div>
          </div>

          <!-- 新建项目无视觉资产时展示电影取景框占位 -->
          <div v-else class="cover-viewfinder">
            <div class="viewfinder-grid" aria-hidden="true" />
            <div class="viewfinder-frame" :class="`ratio-${ratioKey(d)}`">
              <span class="cover-initial">{{ coverInitial(d) }}</span>
              <span class="viewfinder-ratio mono">{{ d.aspect_ratio === 'adaptive' ? 'AUTO' : (d.aspect_ratio || '16:9') }}</span>
            </div>
          </div>

          <div class="cover-scrim" aria-hidden="true" />

          <!-- 状态角标 -->
          <div class="status-wrap" @click.stop>
            <AppMenu
              :open="statusMenuId === d.id"
              placement="bottom-start"
              :min-width="124"
              @update:open="(v) => { statusMenuId = v ? d.id : null }"
            >
              <template #trigger>
                <button type="button" class="cover-badge status-badge" :title="t('index.statusBadgeTitle')">
                  <span class="status-dot" :class="statusDotClass(d)"></span>
                  {{ projectStatus(d) }}
                </button>
              </template>
              <AppMenuItem
                v-for="s in statusOptions"
                :key="s.value"
                :selected="currentStatus(d) === s.value"
                @click="setDramaStatus(d, s.value)"
              >{{ s.label }}</AppMenuItem>
            </AppMenu>
          </div>

          <!-- 更多操作 -->
          <div class="more-wrap" @click.stop>
            <AppMenu
              :open="activeMenuId === d.id"
              placement="bottom-end"
              :min-width="144"
              @update:open="(v) => { activeMenuId = v ? d.id : null }"
            >
              <template #trigger>
                <button class="btn btn-icon btn-sm cover-more" type="button" :title="t('common.more')">
                  <MoreHorizontal :size="15" :stroke-width="2" />
                </button>
              </template>
              <AppMenuItem @click="activeMenuId = null; openDrama(d)">
                <FolderOpen :size="13" class="menu-item-icon" />
                {{ t('index.openProject') }}
              </AppMenuItem>
              <AppMenuItem danger @click="activeMenuId = null; dramaToDelete = d">
                <Trash2 :size="13" class="menu-item-icon" />
                {{ t('index.deleteProject') }}
              </AppMenuItem>
            </AppMenu>
          </div>

          <!-- 底部悬浮标签：画幅 + 风格 -->
          <div class="cover-bottom-bar">
            <span v-if="d.style" class="cover-pill is-accent">{{ styleLabel(d.style) }}</span>
            <span v-if="d.aspect_ratio && d.aspect_ratio !== 'adaptive'" class="cover-pill mono">{{ d.aspect_ratio }}</span>
          </div>
        </div>

        <div class="project-body">
          <div class="project-head-row">
            <h2 class="project-name truncate">{{ d.title }}</h2>
            <span class="project-enter-cue" aria-hidden="true">
              <ArrowUpRight :size="14" :stroke-width="2.1" />
            </span>
          </div>
          <div class="project-foot">
            <span class="project-counts truncate">
              {{ t('index.projectMeta', { chars: d.characters?.length || 0, scenes: d.scenes?.length || 0, eps: d.episodes?.length || 0 }) }}
            </span>
            <span class="project-time">
              <Clock :size="11" :stroke-width="1.8" />
              {{ fmtDate(d.updated_at || d.updatedAt) }}
            </span>
          </div>
        </div>
      </article>
    </div>

    <!-- 空状态 -->
    <div v-else class="empty-state">
      <div class="empty-icon">
        <Film :size="26" :stroke-width="1.4" />
      </div>
      <p class="empty-title">{{ dramas.length ? t('index.emptyFilteredTitle') : t('index.emptyTitle') }}</p>
      <p class="empty-desc">{{ dramas.length ? t('index.emptyFilteredDesc') : t('index.emptyDesc') }}</p>
      <button v-if="!dramas.length" type="button" class="btn btn-primary" @click="showCreate = true">
        <Plus :size="14" :stroke-width="2.2" />
        {{ t('index.create') }}
      </button>
    </div>

    <!-- 新建项目对话框 -->
    <Teleport to="body">
      <div v-if="showCreate" class="overlay" @click.self="showCreate = false">
        <div class="dialog create-dialog">
          <div class="dialog-head">
            <div class="modal-icon">
              <Clapperboard :size="18" :stroke-width="1.8" />
            </div>
            <div class="dialog-head-copy">
              <h2 class="dialog-title">{{ t('index.createDialog.title') }}</h2>
              <p class="dialog-desc">{{ t('index.createDialog.desc') }}</p>
            </div>
            <button type="button" class="btn btn-icon btn-sm btn-ghost ml-auto" :aria-label="t('common.close')" @click="showCreate = false">
              <X :size="15" />
            </button>
          </div>
          <form @submit.prevent="create" class="dialog-form">
            <div class="dialog-body">
              <label class="field">
                <span class="field-label">{{ t('index.createDialog.name') }} <span class="required">*</span></span>
                <input v-model="form.title" class="input" :placeholder="t('index.createDialog.namePlaceholder')" required autofocus />
              </label>
              <div class="field-row">
                <label class="field">
                  <span class="field-label">{{ t('index.createDialog.style') }}</span>
                  <BaseSelect v-model="form.style" :options="styleSelectOptions" :placeholder="t('index.createDialog.stylePlaceholder')" searchable />
                  <span v-if="selectedStyleDesc" class="field-hint">{{ selectedStyleDesc }}</span>
                </label>
                <label class="field">
                  <span class="field-label">{{ t('index.createDialog.aspectRatio') }}</span>
                  <BaseSelect v-model="form.aspect_ratio" :options="aspectRatioOptions" :placeholder="t('index.createDialog.aspectRatioPlaceholder')" />
                  <span class="field-hint">{{ t('index.createDialog.aspectRatioHint') }}</span>
                </label>
              </div>
            </div>
            <div class="dialog-foot">
              <button type="button" class="btn" @click="showCreate = false">{{ t('common.cancel') }}</button>
              <button type="submit" class="btn btn-primary">
                <Plus :size="13" :stroke-width="2.2" />
                {{ t('index.createDialog.submit') }}
              </button>
            </div>
          </form>
        </div>
      </div>
    </Teleport>

    <ConfirmDialog
      :open="!!dramaToDelete"
      :title="t('index.deleteDialog.title')"
      :message="t('index.deleteDialog.message', { title: dramaToDelete?.title })"
      :loading="deletingDrama"
      @confirm="confirmDelDrama"
      @cancel="dramaToDelete = null"
    />
  </div>
</template>

<script setup>
import { toast } from 'vue-sonner'
import { useI18n } from 'vue-i18n'
import {
  ArrowUpRight,
  ChevronRight,
  CircleHelp,
  Clapperboard,
  Clock,
  FileText,
  Film,
  FolderOpen,
  MoreHorizontal,
  Plus,
  Search,
  Sparkles,
  Trash2,
  Users,
  X,
} from 'lucide-vue-next'
import { toastError } from '~/composables/useToast'
import { dramaAPI, stylePresetAPI } from '~/composables/useApi'
import { thumbOf, thumbFallback, posterOf } from '~/composables/useMedia'
import BaseSelect from '~/components/BaseSelect.vue'
import { startTour, autoTour } from '~/composables/useTour'

const { t, locale } = useI18n()

const dramas = ref([])
const loading = ref(false)
const showCreate = ref(false)
const searchKeyword = ref('')
const statusFilter = ref('all')
const sortMode = ref('updated')
const sortOptions = computed(() => ([
  { label: t('index.sortUpdated'), value: 'updated' },
  { label: t('index.sortTitle'), value: 'title' },
]))
const activeMenuId = ref(null)
const dramaToDelete = ref(null)
const deletingDrama = ref(false)
const form = ref({ title: '', style: '', aspect_ratio: '16:9' })
const stylePresets = ref([])
const styleSelectOptions = computed(() => stylePresets.value.map(p => ({ label: p.name, value: p.value })))
const selectedStyleDesc = computed(() => stylePresets.value.find(p => p.value === form.value.style)?.description || '')

const activeCount = computed(() => dramas.value.filter(d => currentStatus(d) === 'active').length)

const aspectRatioOptions = computed(() => ([
  { label: t('index.ratio.landscape'), value: '16:9' },
  { label: t('index.ratio.portrait'), value: '9:16' },
  { label: t('index.ratio.square'), value: '1:1' },
  { label: t('index.ratio.adaptive'), value: 'adaptive' },
]))
const filters = computed(() => ([
  { label: t('index.status.all'), value: 'all' },
  { label: t('index.status.draft'), value: 'draft' },
  { label: t('index.status.active'), value: 'active' },
  { label: t('index.status.completed'), value: 'completed' },
]))
const statusOptions = computed(() => ([
  { label: t('index.status.draft'), value: 'draft' },
  { label: t('index.status.active'), value: 'active' },
  { label: t('index.status.completed'), value: 'completed' },
]))
const statusMenuId = ref(null)

function currentStatus(d) { return d.status || 'draft' }
function projectStatus(d) { return statusOptions.value.find(s => s.value === currentStatus(d))?.label || t('index.status.draft') }
function statusDotClass(d) { return currentStatus(d) === 'active' ? 'on' : currentStatus(d) === 'completed' ? 'done' : '' }

async function setDramaStatus(d, status) {
  statusMenuId.value = null
  if (currentStatus(d) === status) return
  const prev = d.status
  d.status = status
  try {
    await dramaAPI.update(d.id, { status })
  } catch (e) {
    d.status = prev
    toastError(e)
  }
}

function styleLabel(key) {
  return stylePresets.value.find(p => p.value === key)?.name || key || ''
}

function coverInitial(d) {
  return String(d.title || '?').trim().slice(0, 1).toUpperCase() || '?'
}

function ratioKey(d) {
  const r = d.aspect_ratio || '16:9'
  if (r === '9:16') return 'portrait'
  if (r === '1:1') return 'square'
  return 'landscape'
}

function normalizeMediaUrl(raw) {
  if (!raw) return ''
  if (/^https?:\/\//i.test(raw) || raw.startsWith('/')) return raw
  return `/${raw}`
}

// 从剧集已生成的成片海报帧、场景图、角色立绘中提取最多 3 张作为项目海报封面
function projectCovers(d) {
  const list = []
  const seen = new Set()
  const pushImg = (raw, isVideo = false) => {
    const orig = normalizeMediaUrl(raw)
    if (!orig) return
    const thumb = isVideo ? posterOf(orig) : thumbOf(orig)
    if (!thumb || seen.has(thumb)) return
    seen.add(thumb)
    list.push({ thumb, orig: isVideo ? thumb : orig })
  }
  for (const ep of d.episodes || []) {
    const v = ep.video_url || ep.videoUrl
    if (v) pushImg(v, true)
    if (list.length >= 3) return list
  }
  for (const s of d.scenes || []) {
    const img = s.image_url || s.imageUrl
    if (img) pushImg(img, false)
    if (list.length >= 3) return list
  }
  for (const c of d.characters || []) {
    const img = c.image_url || c.imageUrl
    if (img) pushImg(img, false)
    if (list.length >= 3) return list
  }
  return list
}

const filteredDramas = computed(() => {
  const keyword = searchKeyword.value.trim().toLowerCase()
  const items = dramas.value.filter((d) => {
    const text = [d.title, d.style, styleLabel(d.style), projectStatus(d)].filter(Boolean).join(' ').toLowerCase()
    const matchesSearch = !keyword || text.includes(keyword)
    const matchesStatus = statusFilter.value === 'all' || currentStatus(d) === statusFilter.value
    return matchesSearch && matchesStatus
  })

  return [...items].sort((a, b) => {
    if (sortMode.value === 'title') return String(a.title || '').localeCompare(String(b.title || ''), 'zh-CN')
    return new Date(b.updated_at || b.updatedAt || 0).getTime() - new Date(a.updated_at || a.updatedAt || 0).getTime()
  })
})

async function load() {
  loading.value = true
  try {
    const [res, presets] = await Promise.all([dramaAPI.list(), stylePresetAPI.list()])
    dramas.value = res.items || []
    stylePresets.value = presets || []
    if (!form.value.style && stylePresets.value.length) {
      form.value.style = stylePresets.value[0].value
    }
  } catch (e) {
    toastError(e)
  } finally {
    loading.value = false
  }
}

async function create() {
  if (!form.value.title?.trim()) return
  try {
    const d = await dramaAPI.create(form.value)
    showCreate.value = false
    navigateTo(`/drama/${d.id}`)
  } catch (e) {
    toastError(e)
  }
}

async function confirmDelDrama() {
  const d = dramaToDelete.value
  if (!d) return
  try {
    deletingDrama.value = true
    await dramaAPI.del(d.id)
    toast.success(t('index.deleted'))
    dramaToDelete.value = null
    load()
  } catch (e) {
    toastError(e)
  } finally {
    deletingDrama.value = false
  }
}

function getDramaPath(d) {
  return `/drama/${d.id}`
}

function openDrama(d) {
  activeMenuId.value = null
  navigateTo(getDramaPath(d))
}

function fmtDate(s) {
  if (!s) return ''
  const d = new Date(s)
  const now = new Date()
  const diff = now.getTime() - d.getTime()
  if (diff < 60000) return t('index.time.justNow')
  if (diff < 3600000) return t('index.time.minutesAgo', { n: Math.floor(diff / 60000) })
  if (diff < 86400000) return t('index.time.hoursAgo', { n: Math.floor(diff / 3600000) })
  if (diff < 604800000) return t('index.time.daysAgo', { n: Math.floor(diff / 86400000) })
  return d.toLocaleDateString(locale.value === 'zh' ? 'zh-CN' : locale.value, { month: 'short', day: 'numeric' })
}

onMounted(load)

const INDEX_TOUR = [
  { element: '#__nuxt', titleKey: 'tour.index.welcome.title', descKey: 'tour.index.welcome.desc' },
  { element: '.nav-link[href="/settings"]', titleKey: 'tour.index.settings.title', descKey: 'tour.index.settings.desc', popoverSide: 'bottom' },
  { element: '.head-actions .btn-primary', titleKey: 'tour.index.create.title', descKey: 'tour.index.create.desc', popoverSide: 'bottom', popoverAlign: 'end' },
]
onMounted(() => setTimeout(() => autoTour('index', INDEX_TOUR, t), 600))
</script>

<style scoped>
.page {
  padding: var(--sp-5) var(--sp-8) var(--sp-12);
  overflow-y: auto;
  height: 100%;
  display: flex;
  flex-direction: column;
  gap: var(--sp-5);
  animation: fadeUp 0.35s var(--ease-out) both;
  background: var(--surface-base);
}

/* === 造梦工厂 Hero 启动区 === */
.launcher-hero {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-6);
  padding: 24px 28px;
  border-radius: var(--radius-xl);
  border: 1px solid var(--border);
  background: var(--surface-raised);
  box-shadow: var(--shadow-card);
  overflow: hidden;
  flex-shrink: 0;
}

.hero-glow {
  position: absolute;
  inset: 0;
  background: var(--hero-ambient);
  pointer-events: none;
}

.hero-main {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
  min-width: 0;
}

.hero-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  align-self: flex-start;
  padding: 4px 11px;
  border-radius: var(--radius-pill);
  background: var(--accent-bg);
  border: 1px solid var(--accent-glow);
  color: var(--accent-text);
  font-size: 10.5px;
  font-weight: 700;
  letter-spacing: 0.08em;
}

.launcher-title {
  font-size: 26px;
  font-weight: 800;
  letter-spacing: -0.03em;
  color: var(--text-0);
  line-height: 1.2;
}

.launcher-sub {
  color: var(--text-2);
  font-size: 13.5px;
  line-height: 1.5;
  max-width: 540px;
}

.hero-pipeline {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
  margin-top: 4px;
}

.pipe-pill {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 4px 10px;
  border-radius: var(--radius-pill);
  background: var(--bg-2);
  border: 1px solid var(--border);
  color: var(--text-2);
  font-size: 11px;
  font-weight: 600;
}

.pipe-arrow {
  color: var(--text-3);
  flex-shrink: 0;
}

.hero-side {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 14px;
  flex-shrink: 0;
}

.hero-metrics {
  display: flex;
  align-items: center;
  gap: 10px;
}

.metric-tile {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  padding: 8px 14px;
  border-radius: var(--radius);
  background: var(--surface-soft);
  border: 1px solid var(--border);
  min-width: 92px;
}
.metric-tile.is-active { border-color: rgba(16, 185, 129, 0.25); }
.metric-tile.is-accent { border-color: var(--accent-glow); }

.metric-value {
  font-size: 18px;
  font-weight: 750;
  line-height: 1.1;
  color: var(--text-0);
}
.metric-tile.is-active .metric-value { color: var(--success); }
.metric-tile.is-accent .metric-value { color: var(--accent-text); }

.metric-label {
  font-size: 10.5px;
  color: var(--text-3);
  margin-top: 3px;
  white-space: nowrap;
}

.head-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

.hero-create-btn {
  height: 40px;
  padding: 0 22px;
  font-size: 13.5px;
  box-shadow: 0 4px 18px var(--accent-glow);
}

/* 工具栏 */
.toolbar {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  flex-shrink: 0;
}

.search-box {
  position: relative;
  width: 268px;
  flex: 0 0 auto;
}

.search-icon {
  position: absolute;
  left: 14px;
  top: 50%;
  transform: translateY(-50%);
  color: var(--text-3);
  pointer-events: none;
}

.search-box .input {
  padding-left: 38px;
  border-radius: var(--radius-pill);
  border-color: var(--border);
  background: var(--surface-raised);
}
.search-box .input:focus {
  background: var(--surface-input);
}

.filter-seg {
  flex-shrink: 0;
  overflow-x: auto;
}

.sort-select-wrap {
  margin-left: auto;
  width: 144px;
  flex-shrink: 0;
}

/* 项目海报卡片网格 */
.project-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 18px;
}

.project-card {
  position: relative;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  border-radius: var(--radius-lg);
  cursor: pointer;
  animation: fadeUp 0.32s var(--ease-out) both;
  transition: transform 0.22s var(--ease-out), box-shadow 0.22s var(--ease-out), border-color 0.22s var(--ease-out);
}
.project-card:hover {
  transform: translateY(-3px);
  box-shadow: var(--shadow-lift);
  border-color: var(--accent-glow);
}
.project-card:focus-visible {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 3.5px var(--button-focus);
}

/* 封面容器 */
.project-cover {
  position: relative;
  aspect-ratio: 16 / 10;
  overflow: hidden;
  background: var(--bg-2);
  border-bottom: 1px solid var(--border);
}

/* 画报多图拼贴 */
.cover-collage {
  width: 100%;
  height: 100%;
  display: grid;
  gap: 2px;
  background: var(--media-surface);
}
.cover-collage.collage-1 { grid-template-columns: 1fr; }
.cover-collage.collage-2 { grid-template-columns: 1fr 1fr; }
.cover-collage.collage-3 {
  grid-template-columns: 1.35fr 1fr;
  grid-template-rows: 1fr 1fr;
}
.cover-collage.collage-3 .collage-cell:first-child {
  grid-row: 1 / span 2;
}
.collage-cell {
  position: relative;
  overflow: hidden;
  background: var(--bg-2);
}
.collage-cell img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  transition: transform 0.45s var(--ease-out);
}
.project-card:hover .collage-cell img {
  transform: scale(1.06);
}

/* 电影取景框空封面 */
.cover-viewfinder {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background:
    radial-gradient(circle at 25% 20%, var(--accent-bg), transparent 62%),
    radial-gradient(circle at 85% 85%, var(--info-bg), transparent 55%),
    var(--surface-soft);
}
.viewfinder-grid {
  position: absolute;
  inset: 0;
  background-image: radial-gradient(var(--border-strong) 1px, transparent 1px);
  background-size: 16px 16px;
  opacity: 0.45;
}
.viewfinder-frame {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  border: 1.5px dashed var(--border-strong);
  border-radius: var(--radius);
  background: var(--surface-glass);
  backdrop-filter: blur(8px);
  transition: transform 0.25s var(--ease-out), border-color 0.25s var(--ease-out);
}
.viewfinder-frame.ratio-landscape { width: 106px; height: 64px; }
.viewfinder-frame.ratio-portrait { width: 54px; height: 82px; }
.viewfinder-frame.ratio-square { width: 72px; height: 72px; }

.project-card:hover .viewfinder-frame {
  transform: scale(1.05);
  border-color: var(--accent);
}
.cover-initial {
  font-size: 22px;
  font-weight: 800;
  background: var(--accent-gradient);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
  line-height: 1;
  user-select: none;
}
.viewfinder-ratio {
  font-size: 9px;
  font-weight: 700;
  color: var(--text-3);
  letter-spacing: 0.06em;
}

.cover-scrim {
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(8, 9, 14, 0.28) 0%, transparent 38%, rgba(8, 9, 14, 0.56) 100%);
  pointer-events: none;
  opacity: 0.55;
  transition: opacity 0.2s var(--ease-out);
}
.project-cover.has-media .cover-scrim { opacity: 0.85; }

.status-wrap {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 2;
}

.cover-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  background: var(--surface-glass);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  color: var(--text-1);
  font-size: 11px;
  font-weight: 600;
  white-space: nowrap;
  box-shadow: var(--shadow-xs);
}
.status-badge {
  cursor: pointer;
  transition: background 0.14s var(--ease-out), color 0.14s var(--ease-out), border-color 0.14s var(--ease-out);
}
.status-badge:hover {
  color: var(--text-0);
  border-color: var(--border-strong);
}

.status-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: var(--text-3);
}
.status-dot.on {
  background: var(--success);
  box-shadow: 0 0 0 3px var(--success-bg);
}
.status-dot.done { background: var(--accent); }

.more-wrap {
  position: absolute;
  top: 10px;
  right: 10px;
  z-index: 2;
}

.cover-more {
  width: var(--button-height-sm);
  min-width: var(--button-height-sm);
  height: var(--button-height-sm);
  min-height: var(--button-height-sm);
  background: var(--surface-glass);
  backdrop-filter: blur(12px);
  border: 1px solid var(--border);
  color: var(--text-1);
  opacity: 0;
  transition: opacity 0.15s var(--ease-out), color 0.15s var(--ease-out), border-color 0.15s var(--ease-out);
}
.cover-more:hover {
  color: var(--text-0);
  border-color: var(--border-strong);
}
.project-card:hover .cover-more,
.more-wrap:focus-within .cover-more {
  opacity: 1;
}

.cover-bottom-bar {
  position: absolute;
  left: 10px;
  right: 10px;
  bottom: 10px;
  z-index: 2;
  display: flex;
  align-items: center;
  gap: 6px;
  pointer-events: none;
}

.cover-pill {
  padding: 2px 8px;
  border-radius: var(--radius-pill);
  font-size: 10.5px;
  font-weight: 600;
  background: var(--surface-glass);
  backdrop-filter: blur(10px);
  color: var(--text-1);
  border: 1px solid var(--border);
}
.cover-pill.is-accent {
  background: var(--accent-bg);
  color: var(--accent-text);
  border-color: var(--accent-glow);
}

.menu-item-icon {
  flex-shrink: 0;
  opacity: 0.75;
}

/* 卡身 */
.project-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding: 14px 16px;
}

.project-head-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.project-name {
  margin: 0;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: -0.015em;
  color: var(--text-0);
}

.project-enter-cue {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: var(--bg-2);
  color: var(--text-3);
  flex-shrink: 0;
  transition: transform 0.18s var(--ease-out), background 0.18s var(--ease-out), color 0.18s var(--ease-out);
}
.project-card:hover .project-enter-cue {
  transform: translate(1px, -1px);
  background: var(--accent-gradient);
  color: var(--on-accent);
}

.project-foot {
  margin-top: auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--sp-2);
  font-size: 11.5px;
  color: var(--text-3);
}

.project-counts { min-width: 0; }

.project-time {
  display: inline-flex;
  align-items: center;
  gap: var(--sp-1);
  flex-shrink: 0;
  white-space: nowrap;
}

/* 骨架卡片 */
.skeleton-card {
  overflow: hidden;
  border-radius: var(--radius-lg);
}
.skeleton-cover {
  aspect-ratio: 16 / 10;
  background: var(--bg-2);
  animation: skeleton-pulse 1.4s ease-in-out infinite alternate;
}
.skeleton-body {
  padding: var(--sp-4);
  display: grid;
  gap: var(--sp-3);
}
.skeleton-line {
  height: 12px;
  border-radius: var(--radius-pill);
  background: var(--bg-2);
  animation: skeleton-pulse 1.4s ease-in-out infinite alternate;
}
.skeleton-line.w-60 { width: 60%; }
.skeleton-line.w-40 { width: 40%; }
@keyframes skeleton-pulse { to { opacity: 0.5; } }

/* 空状态 */
.empty-state {
  flex: 1;
  min-height: 260px;
  padding: var(--sp-10) var(--sp-6);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: var(--sp-3);
  border: 1px dashed var(--border-strong);
  border-radius: var(--radius-xl);
  background: var(--surface-raised);
  text-align: center;
}
.empty-icon {
  width: 58px;
  height: 58px;
  border-radius: var(--radius-lg);
  background: var(--accent-bg);
  color: var(--accent-text);
  display: flex;
  align-items: center;
  justify-content: center;
}
.empty-title {
  font-size: 15.5px;
  font-weight: 700;
  color: var(--text-0);
}
.empty-desc {
  font-size: 12.5px;
  color: var(--text-3);
  max-width: 300px;
  line-height: 1.6;
}

/* 新建弹窗 */
.create-dialog { width: min(500px, calc(100vw - 32px)); }
.dialog-head-copy { display: flex; flex-direction: column; gap: 2px; min-width: 0; }
.dialog-desc { font-size: 12.5px; color: var(--text-3); }
.modal-icon {
  width: 40px;
  height: 40px;
  flex: 0 0 auto;
  border-radius: var(--radius);
  background: var(--accent-bg);
  color: var(--accent-text);
  display: flex;
  align-items: center;
  justify-content: center;
}
.dialog-form { display: flex; flex-direction: column; flex: 1; min-height: 0; }
.dialog-body { display: flex; flex-direction: column; gap: var(--sp-4); }
.field { display: flex; flex-direction: column; gap: 6px; }
.field-label { font-size: 12px; font-weight: 600; color: var(--text-1); }
.required { color: var(--error); }
.field-hint { font-size: 11.5px; color: var(--text-3); line-height: 1.5; }
.field-row { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: var(--sp-3); }

@media (max-width: 980px) {
  .launcher-hero {
    flex-direction: column;
    align-items: stretch;
    padding: 20px;
  }
  .hero-side {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
  }
}

@media (max-width: 760px) {
  .page { padding: var(--sp-4) var(--sp-4) var(--sp-10); }
  .hero-pipeline,
  .hero-metrics { display: none; }
  .hero-side { justify-content: flex-end; }
  .toolbar { flex-wrap: wrap; }
  .search-box { width: 100%; flex: 1 1 100%; }
  .sort-select-wrap { margin-left: 0; flex: 1; }
  .project-grid { grid-template-columns: repeat(auto-fill, minmax(230px, 1fr)); gap: var(--sp-3); }
  .cover-more { opacity: 1; }
  .field-row { grid-template-columns: 1fr; }
  .dialog-foot { flex-direction: column-reverse; }
  .dialog-foot .btn { width: 100%; }
}
</style>
