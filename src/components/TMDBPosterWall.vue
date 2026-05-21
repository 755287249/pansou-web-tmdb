<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue';
import { search, type HealthStatus, type SearchParams } from '@/api';
import type { MergedResults, SearchResponse } from '@/types';
import ResultTabs from '@/components/ResultTabs.vue';

const TMDB_API_BASE = '/tmdb-api/3';
const TMDB_IMAGE_BASE = 'https://image.tmdb.org/t/p';

type CategoryKey = 'movie' | 'tv' | 'anime' | 'variety';
type ScopeKey = 'cn' | 'global';

type TmdbItem = {
  id: number;
  title?: string;
  name?: string;
  original_title?: string;
  original_name?: string;
  overview?: string;
  poster_path?: string | null;
  backdrop_path?: string | null;
  release_date?: string;
  first_air_date?: string;
  vote_average?: number;
  popularity?: number;
  origin_country?: string[];
  genre_ids?: number[];
};

type WallResponse = {
  page: number;
  total_pages: number;
  total_results: number;
  results: TmdbItem[];
};

type PosterSearchPayloadPreview = {
  kw: string;
  cloud_types: string[];
  plugins: string[];
  channels: string[];
  filter: {
    include: string[];
    exclude: string[];
  };
};

const props = defineProps<{
  backendHealth: HealthStatus | null;
}>();

const categoryMeta: Record<CategoryKey, { label: string; hint: string; icon: string }> = {
  movie: { label: '电影', hint: '院线 / 网络电影', icon: '🎬' },
  tv: { label: '剧集', hint: '电视剧 / 网剧', icon: '📺' },
  anime: { label: '动漫', hint: '动画番剧 / 动画电影', icon: '✨' },
  variety: { label: '综艺', hint: '真人秀 / 脱口秀', icon: '🎤' }
};

const cloudOptions = ['quark', 'uc', 'aliyun', 'baidu', 'xunlei', '115', 'mobile', '123'];
const pluginOptions = ['wanou', 'labi', 'quark4k', 'hunhepan', 'panta', 'fox4k', 'hdr4k', 'pansearch'];
const channelOptions = ['leoziyuan', 'tgsearchers6', 'Quark_Movies', 'Q_dongman', 'q_dianshiju', 'yunpanquark'];

const currentYear = new Date().getFullYear();
const yearOptions = Array.from({ length: 12 }, (_, index) => String(currentYear + 1 - index));

const category = ref<CategoryKey>('movie');
const scope = ref<ScopeKey>('cn');
const selectedYear = ref(String(currentYear));
const query = ref('');
const debouncedQuery = ref('');
const items = ref<TmdbItem[]>([]);
const page = ref(1);
const totalPages = ref(1);
const totalResults = ref(0);
const loadingWall = ref(false);
const wallError = ref('');

const settingsOpen = ref(false);
const selectedCloudTypes = ref<string[]>(['quark']);
const selectedPlugins = ref<string[]>(['wanou', 'labi', 'quark4k']);
const selectedChannels = ref<string[]>(['leoziyuan']);
const includeWords = ref('');
const excludeWords = ref('预告');

const drawerOpen = ref(false);
const selectedItem = ref<TmdbItem | null>(null);
const searching = ref(false);
const isActivelySearching = ref(false);
const searchError = ref('');
const mergedResults = ref<MergedResults>({});
const resultTotal = ref(0);
const resultFilter = ref('');
const hideSuspectedInvalid = ref(false);
const lastPayloadPreview = ref<PosterSearchPayloadPreview | null>(null);
const lastSearchParams = ref<SearchParams | null>(null);
const activeRequestKey = ref(0);
const backgroundTimers = ref<number[]>([]);

const activeCategoryMeta = computed(() => categoryMeta[category.value]);

const visibleItems = computed(() => items.value.filter(item => Boolean(item.poster_path || getTitle(item))));

const hasResults = computed(() => Object.values(mergedResults.value || {}).some(list => Array.isArray(list) && list.length > 0));

const filteredMergedResults = computed<MergedResults>(() => {
  const keyword = resultFilter.value.trim().toLowerCase();
  const next: MergedResults = {};

  Object.entries(mergedResults.value || {}).forEach(([diskType, list]) => {
    if (!Array.isArray(list)) return;

    const filtered = list.filter((item) => {
      const text = `${item.note || ''} ${item.source || ''} ${item.datetime || ''} ${item.url || ''}`.toLowerCase();
      const keywordMatched = keyword ? text.includes(keyword) : true;
      const validMatched = hideSuspectedInvalid.value ? !/失效|过期|无效|取消|不存在|删除/i.test(text) : true;
      return keywordMatched && validMatched;
    });

    if (filtered.length > 0) {
      next[diskType] = filtered;
    }
  });

  return next;
});

const payloadJson = computed(() => JSON.stringify(lastPayloadPreview.value || buildPreviewPayload(null), null, 2));

function splitWords(value: string) {
  return value
    .split(/[，,\s\n]+/)
    .map(item => item.trim())
    .filter(Boolean);
}

function getTitle(item: TmdbItem | null) {
  if (!item) return '';
  return item.title || item.name || item.original_title || item.original_name || `TMDB #${item.id}`;
}

function getYear(item: TmdbItem | null) {
  const date = item?.release_date || item?.first_air_date || '';
  return date ? date.slice(0, 4) : '';
}

function getPosterUrl(path?: string | null, size = 'w342') {
  return path ? `${TMDB_IMAGE_BASE}/${size}${path}` : '';
}

function getBackdropUrl(path?: string | null, size = 'w780') {
  return path ? `${TMDB_IMAGE_BASE}/${size}${path}` : '';
}

function loadPosterConfig() {
  try {
    const cloud = localStorage.getItem('tmdb_pansou_cloud_types');
    const plugins = localStorage.getItem('tmdb_pansou_plugins');
    const channels = localStorage.getItem('tmdb_pansou_channels');
    const include = localStorage.getItem('tmdb_pansou_include_words');
    const exclude = localStorage.getItem('tmdb_pansou_exclude_words');

    if (cloud) selectedCloudTypes.value = JSON.parse(cloud);
    if (plugins) selectedPlugins.value = JSON.parse(plugins);
    if (channels) selectedChannels.value = JSON.parse(channels);
    if (include !== null) includeWords.value = include;
    if (exclude !== null) excludeWords.value = exclude;
  } catch (error) {
    console.warn('读取海报墙配置失败，使用默认配置', error);
  }
}

function savePosterConfig() {
  localStorage.setItem('tmdb_pansou_cloud_types', JSON.stringify(selectedCloudTypes.value));
  localStorage.setItem('tmdb_pansou_plugins', JSON.stringify(selectedPlugins.value));
  localStorage.setItem('tmdb_pansou_channels', JSON.stringify(selectedChannels.value));
  localStorage.setItem('tmdb_pansou_include_words', includeWords.value);
  localStorage.setItem('tmdb_pansou_exclude_words', excludeWords.value);
}

function useMainSearchConfig() {
  try {
    const savedCloudTypes = localStorage.getItem('pansou_disk_types');
    const savedPlugins = localStorage.getItem('pansou_plugins');
    const savedChannels = localStorage.getItem('pansou_channels');

    if (savedCloudTypes) selectedCloudTypes.value = JSON.parse(savedCloudTypes);
    if (savedPlugins) selectedPlugins.value = JSON.parse(savedPlugins);
    if (savedChannels) selectedChannels.value = JSON.parse(savedChannels);

    if (!savedPlugins && props.backendHealth?.plugins?.length) {
      selectedPlugins.value = props.backendHealth.plugins;
    }
    if (!savedChannels && props.backendHealth?.channels?.length) {
      selectedChannels.value = props.backendHealth.channels;
    }

    savePosterConfig();
  } catch (error) {
    console.error('同步主配置失败:', error);
  }
}

function toggleValue(list: string[], value: string) {
  const index = list.indexOf(value);
  if (index >= 0) {
    list.splice(index, 1);
  } else {
    list.push(value);
  }
  savePosterConfig();
}

function buildTmdbUrl(targetPage = 1) {
  const params = new URLSearchParams({
    language: 'zh-CN',
    include_adult: 'false',
    page: String(targetPage)
  });

  let endpoint = '';
  const trimmedQuery = debouncedQuery.value.trim();
  const isMovie = category.value === 'movie';

  if (trimmedQuery) {
    endpoint = isMovie ? '/search/movie' : '/search/tv';
    params.set('query', trimmedQuery);
    if (isMovie) {
      params.set('year', selectedYear.value);
    } else {
      params.set('first_air_date_year', selectedYear.value);
    }
  } else {
    endpoint = isMovie ? '/discover/movie' : '/discover/tv';
    params.set('sort_by', 'popularity.desc');

    if (isMovie) {
      params.set('primary_release_year', selectedYear.value);
    } else {
      params.set('first_air_date_year', selectedYear.value);
    }

    if (category.value === 'anime') {
      params.set('with_genres', '16');
    }

    if (category.value === 'variety') {
      params.set('with_genres', '10764|10767');
    }
  }

  if (scope.value === 'cn') {
    params.set('with_origin_country', 'CN');
    params.set('region', 'CN');
  }

  return `${TMDB_API_BASE}${endpoint}?${params.toString()}`;
}

async function fetchWall(targetPage = 1, append = false) {
  const requestKey = ++activeRequestKey.value;
  loadingWall.value = true;
  wallError.value = '';

  try {
    const response = await fetch(buildTmdbUrl(targetPage));
    const data = await response.json();

    if (!response.ok) {
      throw new Error(data?.status_message || data?.message || `TMDB 请求失败：HTTP ${response.status}`);
    }

    if (requestKey !== activeRequestKey.value) return;

    const wallResponse = data as WallResponse;
    const nextItems = Array.isArray(wallResponse.results) ? wallResponse.results : [];

    items.value = append ? [...items.value, ...nextItems] : nextItems;
    page.value = Number(wallResponse.page || targetPage);
    totalPages.value = Math.min(Number(wallResponse.total_pages || 1), 500);
    totalResults.value = Number(wallResponse.total_results || nextItems.length);
  } catch (error) {
    if (requestKey !== activeRequestKey.value) return;
    wallError.value = error instanceof Error ? error.message : 'TMDB 海报墙加载失败';
    if (!append) items.value = [];
  } finally {
    if (requestKey === activeRequestKey.value) {
      loadingWall.value = false;
    }
  }
}

function buildPreviewPayload(item: TmdbItem | null): PosterSearchPayloadPreview {
  const title = item ? getTitle(item) : '主角';
  const itemYear = (item ? getYear(item) : '') || selectedYear.value;
  const include = Array.from(new Set([itemYear, ...splitWords(includeWords.value)].filter(Boolean)));
  const exclude = Array.from(new Set(splitWords(excludeWords.value)));

  return {
    kw: title,
    cloud_types: [...selectedCloudTypes.value],
    plugins: [...selectedPlugins.value],
    channels: [...selectedChannels.value],
    filter: {
      include,
      exclude
    }
  };
}

function buildSearchParams(item: TmdbItem, srcOverride?: 'all' | 'tg' | 'plugin'): SearchParams {
  const payload = buildPreviewPayload(item);
  const hasChannels = payload.channels.length > 0;
  const hasPlugins = payload.plugins.length > 0;

  let src: 'all' | 'tg' | 'plugin' = 'all';
  if (srcOverride) {
    src = srcOverride;
  } else if (hasChannels && hasPlugins) {
    src = 'tg';
  } else if (hasChannels) {
    src = 'tg';
  } else if (hasPlugins) {
    src = 'plugin';
  }

  const params: SearchParams = {
    kw: payload.kw,
    res: 'merge',
    src
  };

  if (payload.channels.length) {
    (params as any).channels = payload.channels.join(',');
  }
  if (payload.plugins.length) {
    params.plugins = payload.plugins.join(',');
  }
  if (payload.cloud_types.length) {
    (params as any).cloud_types = payload.cloud_types.join(',');
  }
  if (payload.filter.include.length || payload.filter.exclude.length) {
    (params as any).filter = JSON.stringify(payload.filter);
  }

  lastPayloadPreview.value = payload;
  return params;
}

function mergeSearchResponse(response: SearchResponse, replace = false) {
  resultTotal.value = response.total || 0;

  const incoming = response.merged_by_type || {};
  if (replace) {
    mergedResults.value = incoming;
    return;
  }

  const next: MergedResults = { ...mergedResults.value };

  Object.entries(incoming).forEach(([diskType, list]) => {
    if (!Array.isArray(list)) return;

    const existing = next[diskType] || [];
    const seen = new Set(existing.map(item => item.url || `${item.note}-${item.source}`));
    const additions = list.filter((item) => {
      const key = item.url || `${item.note}-${item.source}`;
      if (seen.has(key)) return false;
      seen.add(key);
      return true;
    });

    next[diskType] = [...existing, ...additions];
  });

  mergedResults.value = next;
}

function clearBackgroundTimers() {
  backgroundTimers.value.forEach(timer => window.clearTimeout(timer));
  backgroundTimers.value = [];
}

function scheduleBackgroundRefresh(item: TmdbItem) {
  clearBackgroundTimers();

  if (!selectedPlugins.value.length) {
    isActivelySearching.value = false;
    return;
  }

  const hasChannels = selectedChannels.value.length > 0;
  const src: 'all' | 'plugin' = hasChannels ? 'all' : 'plugin';

  [2500, 6000].forEach((delay) => {
    const timer = window.setTimeout(async () => {
      try {
        const params = buildSearchParams(item, src);
        const response = await search(params);
        mergeSearchResponse(response);
      } catch (error) {
        console.warn('海报墙后台补充搜索失败:', error);
      } finally {
        if (delay === 6000) {
          isActivelySearching.value = false;
        }
      }
    }, delay);
    backgroundTimers.value.push(timer);
  });
}

async function searchFromPoster(item: TmdbItem) {
  clearBackgroundTimers();
  selectedItem.value = item;
  drawerOpen.value = true;
  searching.value = true;
  isActivelySearching.value = true;
  searchError.value = '';
  resultFilter.value = '';
  mergedResults.value = {};
  resultTotal.value = 0;

  try {
    const params = buildSearchParams(item);
    lastSearchParams.value = params;
    const response = await search(params);
    mergeSearchResponse(response, true);
    searching.value = false;
    scheduleBackgroundRefresh(item);
  } catch (error) {
    searching.value = false;
    isActivelySearching.value = false;
    searchError.value = error instanceof Error ? error.message : 'PanSou 搜索失败';
  }
}

async function retrySearch() {
  if (!selectedItem.value) return;
  await searchFromPoster(selectedItem.value);
}

async function copyPayload() {
  try {
    await navigator.clipboard.writeText(payloadJson.value);
  } catch (error) {
    console.warn('复制请求体失败:', error);
  }
}

function closeDrawer() {
  drawerOpen.value = false;
  clearBackgroundTimers();
  isActivelySearching.value = false;
}

let debounceTimer: number | undefined;
watch(query, () => {
  if (debounceTimer) window.clearTimeout(debounceTimer);
  debounceTimer = window.setTimeout(() => {
    debouncedQuery.value = query.value.trim();
  }, 350);
});

watch([category, scope, selectedYear, debouncedQuery], () => {
  page.value = 1;
  fetchWall(1, false);
});

onMounted(() => {
  loadPosterConfig();
  fetchWall(1, false);
});

onUnmounted(() => {
  clearBackgroundTimers();
  if (debounceTimer) window.clearTimeout(debounceTimer);
});
</script>

<template>
  <div class="poster-wall-page">
    <section class="poster-hero">
      <div class="poster-hero-content">
        <div class="poster-eyebrow">TMDB Poster Wall × PanSou</div>
        <h2>海报墙资源搜索</h2>
        <p>从 TMDB 拉取电影、剧集、动漫、综艺海报；点击海报后自动用标题作为 kw，并把年份加入 PanSou 过滤条件。</p>
      </div>
      <div class="poster-hero-actions">
        <div class="scope-switch">
          <button :class="{ active: scope === 'cn' }" @click="scope = 'cn'">国产</button>
          <button :class="{ active: scope === 'global' }" @click="scope = 'global'">全球</button>
        </div>
        <select v-model="selectedYear" class="poster-select">
          <option v-for="year in yearOptions" :key="year" :value="year">{{ year }}</option>
        </select>
        <button class="poster-outline-button" @click="settingsOpen = true">搜索配置</button>
      </div>
    </section>

    <section class="poster-toolbar">
      <div class="category-tabs">
        <button
          v-for="(meta, key) in categoryMeta"
          :key="key"
          :class="['category-card', { active: category === key }]"
          @click="category = key as CategoryKey"
        >
          <span class="category-icon">{{ meta.icon }}</span>
          <span>
            <strong>{{ meta.label }}</strong>
            <em>{{ meta.hint }}</em>
          </span>
        </button>
      </div>
      <div class="poster-search-input-wrap">
        <input
          v-model="query"
          class="poster-search-input"
          :placeholder="`搜索${activeCategoryMeta.label}标题`"
          @keydown.enter="debouncedQuery = query.trim()"
        />
      </div>
    </section>

    <section class="poster-section-header">
      <div>
        <h3>{{ selectedYear }} · {{ scope === 'cn' ? '国产' : '全球' }} · {{ activeCategoryMeta.label }}</h3>
        <p>当前默认请求：cloud_types={{ selectedCloudTypes.join(',') || '未选择' }}；plugins={{ selectedPlugins.join(',') || '未选择' }}；channels={{ selectedChannels.join(',') || '未选择' }}</p>
      </div>
      <button class="poster-outline-button" :disabled="loadingWall" @click="fetchWall(1, false)">
        {{ loadingWall ? '加载中...' : '刷新海报墙' }}
      </button>
    </section>

    <div v-if="loadingWall && !items.length" class="poster-grid">
      <div v-for="index in 18" :key="index" class="poster-skeleton">
        <div></div>
        <span></span>
        <i></i>
      </div>
    </div>

    <div v-else-if="wallError" class="poster-empty">
      <strong>海报墙加载失败</strong>
      <p>{{ wallError }}</p>
      <p class="poster-empty-hint">请确认容器环境变量 TMDB_BEARER_TOKEN 已配置，或开发环境 Vite 代理已配置 TMDB Token。</p>
      <button class="poster-primary-button" @click="fetchWall(1, false)">重新加载</button>
    </div>

    <div v-else-if="visibleItems.length" class="poster-grid">
      <button
        v-for="item in visibleItems"
        :key="`${category}-${item.id}`"
        class="poster-card"
        @click="searchFromPoster(item)"
      >
        <div class="poster-image-wrap">
          <img v-if="item.poster_path" :src="getPosterUrl(item.poster_path)" :alt="getTitle(item)" loading="lazy" />
          <div v-else class="poster-no-image">无海报</div>
          <span v-if="item.vote_average" class="poster-rating">{{ item.vote_average.toFixed(1) }}</span>
          <span class="poster-hover-tip">点击搜索网盘资源</span>
        </div>
        <div class="poster-card-body">
          <strong>{{ getTitle(item) }}</strong>
          <span>{{ getYear(item) || '未知年份' }} · {{ item.origin_country?.join('/') || 'TMDB' }}</span>
        </div>
      </button>
    </div>

    <div v-else class="poster-empty">
      <strong>暂无海报内容</strong>
      <p>可以切换分类、年份、国产/全球，或输入标题搜索。</p>
    </div>

    <div v-if="visibleItems.length" class="poster-load-more">
      <span>TMDB 共 {{ totalResults }} 条，当前第 {{ page }} / {{ totalPages }} 页</span>
      <button class="poster-primary-button" :disabled="loadingWall || page >= totalPages" @click="fetchWall(page + 1, true)">
        {{ page >= totalPages ? '没有更多了' : loadingWall ? '加载中...' : '加载更多' }}
      </button>
    </div>

    <Teleport to="body">
      <div v-if="settingsOpen" class="poster-modal-mask" @click="settingsOpen = false">
        <aside class="poster-settings-panel" @click.stop>
          <div class="poster-panel-header">
            <div>
              <h3>海报墙搜索配置</h3>
              <p>这些配置只影响海报墙点击搜索，不会覆盖主搜索页配置。</p>
            </div>
            <button @click="settingsOpen = false">×</button>
          </div>

          <div class="poster-panel-content">
            <section>
              <div class="poster-setting-title">网盘类型 cloud_types</div>
              <div class="poster-chip-list">
                <button
                  v-for="option in cloudOptions"
                  :key="option"
                  :class="['poster-chip', { active: selectedCloudTypes.includes(option) }]"
                  @click="toggleValue(selectedCloudTypes, option)"
                >
                  {{ option }}
                </button>
              </div>
            </section>

            <section>
              <div class="poster-setting-title">插件 plugins</div>
              <div class="poster-chip-list">
                <button
                  v-for="option in pluginOptions"
                  :key="option"
                  :class="['poster-chip', { active: selectedPlugins.includes(option) }]"
                  @click="toggleValue(selectedPlugins, option)"
                >
                  {{ option }}
                </button>
              </div>
            </section>

            <section>
              <div class="poster-setting-title">频道 channels</div>
              <div class="poster-chip-list">
                <button
                  v-for="option in channelOptions"
                  :key="option"
                  :class="['poster-chip', { active: selectedChannels.includes(option) }]"
                  @click="toggleValue(selectedChannels, option)"
                >
                  {{ option }}
                </button>
              </div>
            </section>

            <section>
              <div class="poster-setting-title">过滤条件 filter</div>
              <label class="poster-field">
                <span>额外包含词 include，年份会自动加入</span>
                <input v-model="includeWords" @blur="savePosterConfig" placeholder="例如：4K, 国语, 2160p" />
              </label>
              <label class="poster-field">
                <span>排除词 exclude</span>
                <input v-model="excludeWords" @blur="savePosterConfig" placeholder="例如：预告, 花絮, 解说" />
              </label>
            </section>

            <section class="poster-preview-box">
              <div class="poster-setting-title">请求体预览</div>
              <pre>{{ JSON.stringify(buildPreviewPayload(null), null, 2) }}</pre>
            </section>
          </div>

          <div class="poster-panel-footer">
            <button class="poster-outline-button" @click="useMainSearchConfig">同步主搜索配置</button>
            <button class="poster-primary-button" @click="settingsOpen = false">完成</button>
          </div>
        </aside>
      </div>
    </Teleport>

    <Teleport to="body">
      <div v-if="drawerOpen" class="poster-modal-mask result-mask" @click="closeDrawer">
        <section class="poster-result-drawer" @click.stop>
          <div
            class="result-hero"
            :style="selectedItem?.backdrop_path ? { backgroundImage: `linear-gradient(90deg, rgba(15,23,42,.92), rgba(15,23,42,.58)), url(${getBackdropUrl(selectedItem.backdrop_path)})` } : undefined"
          >
            <div class="result-hero-main">
              <img v-if="selectedItem?.poster_path" :src="getPosterUrl(selectedItem.poster_path, 'w185')" :alt="getTitle(selectedItem)" />
              <div>
                <span>PanSou 搜索结果</span>
                <h3>{{ getTitle(selectedItem) }}</h3>
                <p>{{ getYear(selectedItem) || selectedYear }} · {{ resultTotal }} 条结果</p>
              </div>
            </div>
            <div class="result-hero-actions">
              <button @click="copyPayload">复制请求体</button>
              <button @click="retrySearch">重搜</button>
              <button class="result-close" @click="closeDrawer">×</button>
            </div>
          </div>

          <div class="result-tools">
            <input v-model="resultFilter" placeholder="在当前显示结果里筛选标题、来源、时间、链接" />
            <button :class="{ active: hideSuspectedInvalid }" @click="hideSuspectedInvalid = !hideSuspectedInvalid">
              隐藏疑似失效
            </button>
          </div>

          <div class="result-content">
            <aside class="result-payload">
              <div class="poster-setting-title">实际搜索参数</div>
              <pre>{{ payloadJson }}</pre>
              <p>说明：前端会以 GET /api/search 调用原 PanSou API，并把年份放入 filter.include。</p>
            </aside>

            <main class="result-list">
              <div v-if="searchError" class="poster-empty compact">
                <strong>搜索失败</strong>
                <p>{{ searchError }}</p>
                <button class="poster-primary-button" @click="retrySearch">重新搜索</button>
              </div>

              <div v-else-if="searching && !hasResults" class="poster-empty compact">
                <strong>正在搜索...</strong>
                <p>正在触发 PanSou API；如果启用了插件，会继续在后台补充结果。</p>
              </div>

              <ResultTabs
                v-else
                :merged-results="filteredMergedResults"
                :loading="searching"
                :has-searched="Boolean(selectedItem)"
                :is-actively-searching="isActivelySearching"
              />
            </main>
          </div>
        </section>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
.poster-wall-page {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.poster-hero {
  display: flex;
  align-items: stretch;
  justify-content: space-between;
  gap: 1rem;
  padding: 1.5rem;
  border: 1px solid hsl(var(--border));
  border-radius: 1.5rem;
  background:
    radial-gradient(circle at top left, hsl(var(--primary) / .14), transparent 32%),
    linear-gradient(135deg, hsl(var(--card)), hsl(var(--muted) / .42));
  box-shadow: 0 18px 45px rgba(15, 23, 42, .06);
}

.poster-eyebrow {
  display: inline-flex;
  width: fit-content;
  padding: .35rem .65rem;
  border-radius: 999px;
  background: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
  font-size: .75rem;
  font-weight: 700;
}

.poster-hero h2 {
  margin-top: .8rem;
  font-size: clamp(1.7rem, 4vw, 2.55rem);
  line-height: 1.05;
  font-weight: 800;
  letter-spacing: -.04em;
}

.poster-hero p {
  max-width: 42rem;
  margin-top: .6rem;
  color: hsl(var(--muted-foreground));
  line-height: 1.7;
}

.poster-hero-actions,
.poster-section-header {
  display: flex;
  align-items: center;
  gap: .75rem;
  flex-wrap: wrap;
}

.poster-hero-actions {
  justify-content: flex-end;
  align-content: flex-start;
}

.scope-switch {
  display: inline-flex;
  gap: .25rem;
  padding: .25rem;
  border-radius: .9rem;
  border: 1px solid hsl(var(--border));
  background: hsl(var(--background));
}

.scope-switch button,
.poster-outline-button,
.poster-primary-button,
.poster-chip,
.result-hero-actions button,
.result-tools button {
  border: none;
  cursor: pointer;
  transition: all .2s ease;
}

.scope-switch button {
  padding: .55rem .85rem;
  border-radius: .7rem;
  color: hsl(var(--muted-foreground));
  background: transparent;
  font-weight: 700;
}

.scope-switch button.active,
.poster-chip.active,
.result-tools button.active {
  background: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
}

.poster-select,
.poster-search-input,
.poster-field input,
.result-tools input {
  border: 1px solid hsl(var(--border));
  background: hsl(var(--background));
  color: hsl(var(--foreground));
  border-radius: .9rem;
  outline: none;
  transition: all .2s ease;
}

.poster-select {
  height: 2.75rem;
  padding: 0 .85rem;
}

.poster-search-input:focus,
.poster-field input:focus,
.result-tools input:focus,
.poster-select:focus {
  border-color: hsl(var(--primary));
  box-shadow: 0 0 0 4px hsl(var(--primary) / .12);
}

.poster-toolbar {
  display: grid;
  grid-template-columns: 1fr minmax(240px, 360px);
  gap: 1rem;
  align-items: center;
}

.category-tabs {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: .75rem;
}

.category-card {
  display: flex;
  align-items: center;
  gap: .7rem;
  padding: .9rem;
  border: 1px solid hsl(var(--border));
  border-radius: 1rem;
  background: hsl(var(--card));
  color: hsl(var(--foreground));
  text-align: left;
  cursor: pointer;
  transition: all .2s ease;
}

.category-card:hover,
.poster-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 18px 35px rgba(15, 23, 42, .08);
}

.category-card.active {
  background: hsl(var(--primary));
  border-color: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
}

.category-icon {
  font-size: 1.5rem;
}

.category-card strong,
.category-card em {
  display: block;
}

.category-card em {
  margin-top: .15rem;
  font-size: .75rem;
  font-style: normal;
  opacity: .72;
}

.poster-search-input-wrap {
  position: relative;
}

.poster-search-input {
  width: 100%;
  height: 3.25rem;
  padding: 0 1rem;
}

.poster-section-header {
  justify-content: space-between;
}

.poster-section-header h3 {
  font-size: 1.25rem;
  font-weight: 800;
}

.poster-section-header p {
  margin-top: .25rem;
  color: hsl(var(--muted-foreground));
  font-size: .875rem;
}

.poster-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(145px, 1fr));
  gap: 1rem;
}

.poster-card {
  overflow: hidden;
  border: 1px solid hsl(var(--border));
  border-radius: 1.1rem;
  background: hsl(var(--card));
  color: hsl(var(--foreground));
  text-align: left;
  box-shadow: 0 6px 20px rgba(15, 23, 42, .04);
  transition: all .22s ease;
}

.poster-image-wrap {
  position: relative;
  aspect-ratio: 2 / 3;
  overflow: hidden;
  background: hsl(var(--muted));
}

.poster-image-wrap img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: transform .35s ease;
}

.poster-card:hover img {
  transform: scale(1.05);
}

.poster-rating {
  position: absolute;
  right: .55rem;
  top: .55rem;
  padding: .2rem .45rem;
  border-radius: 999px;
  background: rgba(15, 23, 42, .72);
  color: white;
  font-size: .75rem;
  font-weight: 800;
}

.poster-hover-tip {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  padding: 2rem .75rem .75rem;
  background: linear-gradient(transparent, rgba(15, 23, 42, .88));
  color: white;
  font-size: .8rem;
  font-weight: 700;
  opacity: 0;
  transform: translateY(8px);
  transition: all .2s ease;
}

.poster-card:hover .poster-hover-tip {
  opacity: 1;
  transform: translateY(0);
}

.poster-no-image {
  display: grid;
  width: 100%;
  height: 100%;
  place-items: center;
  color: hsl(var(--muted-foreground));
}

.poster-card-body {
  padding: .8rem;
}

.poster-card-body strong {
  display: -webkit-box;
  min-height: 2.6rem;
  overflow: hidden;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  font-size: .92rem;
  line-height: 1.35;
}

.poster-card-body span {
  display: block;
  margin-top: .45rem;
  color: hsl(var(--muted-foreground));
  font-size: .75rem;
}

.poster-skeleton {
  overflow: hidden;
  border: 1px solid hsl(var(--border));
  border-radius: 1.1rem;
  background: hsl(var(--card));
}

.poster-skeleton div {
  aspect-ratio: 2 / 3;
  background: hsl(var(--muted));
  animation: poster-pulse 1.2s infinite ease-in-out;
}

.poster-skeleton span,
.poster-skeleton i {
  display: block;
  height: .8rem;
  margin: .75rem;
  border-radius: 999px;
  background: hsl(var(--muted));
  animation: poster-pulse 1.2s infinite ease-in-out;
}

.poster-skeleton i {
  width: 45%;
  margin-top: 0;
}

@keyframes poster-pulse {
  0%, 100% { opacity: .55; }
  50% { opacity: 1; }
}

.poster-empty {
  display: flex;
  min-height: 18rem;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: .7rem;
  border: 1px dashed hsl(var(--border));
  border-radius: 1.5rem;
  background: hsl(var(--card) / .75);
  text-align: center;
  padding: 2rem;
}

.poster-empty.compact {
  min-height: 14rem;
}

.poster-empty strong {
  font-size: 1.1rem;
}

.poster-empty p {
  color: hsl(var(--muted-foreground));
}

.poster-empty-hint {
  max-width: 42rem;
  font-size: .85rem;
}

.poster-load-more {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 1rem;
  color: hsl(var(--muted-foreground));
  font-size: .875rem;
}

.poster-outline-button,
.poster-primary-button {
  min-height: 2.65rem;
  padding: .65rem 1rem;
  border-radius: .9rem;
  font-weight: 800;
}

.poster-outline-button {
  border: 1px solid hsl(var(--border));
  background: hsl(var(--background));
  color: hsl(var(--foreground));
}

.poster-outline-button:hover {
  border-color: hsl(var(--primary));
  color: hsl(var(--primary));
}

.poster-primary-button {
  background: hsl(var(--primary));
  color: hsl(var(--primary-foreground));
}

.poster-primary-button:disabled,
.poster-outline-button:disabled {
  opacity: .55;
  cursor: not-allowed;
}

.poster-modal-mask {
  position: fixed;
  inset: 0;
  z-index: 100;
  display: flex;
  justify-content: flex-end;
  background: rgba(15, 23, 42, .42);
  backdrop-filter: blur(8px);
}

.poster-settings-panel {
  width: min(100vw, 440px);
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: hsl(var(--background));
  box-shadow: -20px 0 45px rgba(15, 23, 42, .18);
}

.poster-panel-header,
.poster-panel-footer {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem;
  border-bottom: 1px solid hsl(var(--border));
}

.poster-panel-footer {
  border-top: 1px solid hsl(var(--border));
  border-bottom: none;
}

.poster-panel-header h3 {
  font-size: 1.1rem;
  font-weight: 800;
}

.poster-panel-header p {
  margin-top: .25rem;
  color: hsl(var(--muted-foreground));
  font-size: .82rem;
}

.poster-panel-header button,
.result-close {
  width: 2.3rem;
  height: 2.3rem;
  border-radius: 999px;
  border: 1px solid hsl(var(--border));
  background: hsl(var(--card));
  color: hsl(var(--foreground));
  cursor: pointer;
  font-size: 1.4rem;
}

.poster-panel-content {
  flex: 1;
  overflow: auto;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1.2rem;
}

.poster-setting-title {
  margin-bottom: .65rem;
  font-size: .9rem;
  font-weight: 800;
}

.poster-chip-list {
  display: flex;
  flex-wrap: wrap;
  gap: .5rem;
}

.poster-chip {
  padding: .45rem .7rem;
  border: 1px solid hsl(var(--border));
  border-radius: 999px;
  background: hsl(var(--card));
  color: hsl(var(--muted-foreground));
  font-size: .8rem;
  font-weight: 800;
}

.poster-field {
  display: flex;
  flex-direction: column;
  gap: .4rem;
  margin-bottom: .75rem;
}

.poster-field span {
  color: hsl(var(--muted-foreground));
  font-size: .8rem;
}

.poster-field input {
  min-height: 2.75rem;
  padding: 0 .85rem;
}

.poster-preview-box,
.result-payload {
  border: 1px solid hsl(var(--border));
  border-radius: 1rem;
  background: hsl(var(--muted) / .38);
  padding: 1rem;
}

.poster-preview-box pre,
.result-payload pre {
  overflow: auto;
  max-height: 18rem;
  border-radius: .85rem;
  background: #0f172a;
  color: #e2e8f0;
  padding: .85rem;
  font-size: .78rem;
  line-height: 1.6;
}

.result-mask {
  align-items: flex-end;
  justify-content: center;
}

.poster-result-drawer {
  width: min(1180px, calc(100vw - 1.5rem));
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  border-radius: 1.5rem 1.5rem 0 0;
  background: hsl(var(--background));
  box-shadow: 0 -18px 55px rgba(15, 23, 42, .22);
}

.result-hero {
  display: flex;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem;
  background: linear-gradient(90deg, #0f172a, #1e293b);
  color: white;
  background-size: cover;
  background-position: center;
}

.result-hero-main {
  display: flex;
  align-items: center;
  gap: .9rem;
  min-width: 0;
}

.result-hero-main img {
  width: 4rem;
  height: 6rem;
  object-fit: cover;
  border-radius: .7rem;
  box-shadow: 0 10px 24px rgba(0, 0, 0, .35);
}

.result-hero-main span {
  color: rgba(255,255,255,.68);
  font-size: .8rem;
  font-weight: 800;
}

.result-hero-main h3 {
  margin-top: .2rem;
  font-size: 1.35rem;
  font-weight: 900;
}

.result-hero-main p {
  margin-top: .2rem;
  color: rgba(255,255,255,.75);
}

.result-hero-actions {
  display: flex;
  align-items: flex-start;
  gap: .5rem;
  flex-wrap: wrap;
  justify-content: flex-end;
}

.result-hero-actions button {
  min-height: 2.35rem;
  border-radius: .8rem;
  padding: .45rem .8rem;
  background: rgba(255,255,255,.12);
  color: white;
  font-weight: 800;
}

.result-tools {
  display: flex;
  gap: .75rem;
  padding: .8rem 1rem;
  border-bottom: 1px solid hsl(var(--border));
}

.result-tools input {
  flex: 1;
  min-height: 2.65rem;
  padding: 0 .9rem;
}

.result-tools button {
  border-radius: .9rem;
  padding: .55rem .9rem;
  border: 1px solid hsl(var(--border));
  background: hsl(var(--card));
  color: hsl(var(--foreground));
  font-weight: 800;
}

.result-content {
  min-height: 0;
  flex: 1;
  display: grid;
  grid-template-columns: 310px minmax(0, 1fr);
}

.result-payload {
  border-radius: 0;
  border-top: none;
  border-left: none;
  border-bottom: none;
  overflow: auto;
}

.result-payload p {
  margin-top: .8rem;
  color: hsl(var(--muted-foreground));
  font-size: .8rem;
  line-height: 1.6;
}

.result-list {
  min-height: 0;
  overflow: auto;
  padding: 1rem;
}

@media (max-width: 900px) {
  .poster-hero,
  .poster-section-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .poster-toolbar {
    grid-template-columns: 1fr;
  }

  .category-tabs {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .result-content {
    grid-template-columns: 1fr;
  }

  .result-payload {
    display: none;
  }
}

@media (max-width: 640px) {
  .poster-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .poster-hero,
  .poster-toolbar,
  .poster-section-header {
    gap: .75rem;
  }

  .poster-result-drawer {
    width: 100vw;
    max-height: 94vh;
    border-radius: 1rem 1rem 0 0;
  }

  .result-hero,
  .result-tools,
  .poster-load-more {
    flex-direction: column;
    align-items: stretch;
  }
}
</style>
