<template>
  <section class="brief-page">
    <div class="brief-heading">
      <div>
        <n-tag type="error" size="small" round>AI DAILY BRIEF</n-tag>
        <h1>每日简报</h1>
        <p>把科技、财经与全球热点浓缩成一份可读的晨报。</p>
      </div>
      <n-button secondary strong round :loading="loading" @click="loadBrief">
        重新获取
      </n-button>
    </div>

    <n-alert v-if="error" type="warning" :show-icon="false" class="brief-alert">
      {{ error }}
      <template #action>
        <n-button text type="primary" @click="loadBrief">重试</n-button>
      </template>
    </n-alert>

    <n-spin :show="loading">
      <template v-if="report">
        <n-card :bordered="false" class="hero-card">
          <div class="report-meta">{{ reportDateLabel }}</div>
          <h2>{{ report.hero_headline || "今日重点" }}</h2>
          <p>{{ report.daily_overview }}</p>
          <div v-if="report.keywords?.length" class="keywords">
            <n-tag v-for="keyword in report.keywords" :key="keyword" round size="small">
              {{ keyword }}
            </n-tag>
          </div>
        </n-card>

        <div class="brief-grid">
          <BriefSection title="科技要闻" :items="report.tech_briefs" />
          <BriefSection title="财经动态" :items="report.finance_briefs" />
          <BriefSection title="全球视野" :items="report.politics_briefs" />
        </div>

        <n-card v-if="report.trading" :bordered="false" class="trading-card">
          <div class="section-title">
            <h3>市场观察</h3>
            <span>{{ report.trading.generated_at || "AI 生成" }}</span>
          </div>
          <p v-if="report.trading.overview">{{ report.trading.overview }}</p>
          <div v-if="report.trading.tickers?.length" class="ticker-list">
            <div v-for="ticker in report.trading.tickers.slice(0, 8)" :key="ticker.symbol" class="ticker">
              <strong>{{ ticker.symbol }}</strong>
              <span>{{ ticker.name || "" }}</span>
              <n-tag size="small" :type="ticker.change_pct >= 0 ? 'success' : 'error'">
                {{ formatChange(ticker.change_pct) }}
              </n-tag>
            </div>
          </div>
        </n-card>

        <n-card v-if="report.editor_note" :bordered="false" class="editor-note">
          <h3>编辑手记</h3>
          <p>{{ report.editor_note }}</p>
        </n-card>
      </template>
    </n-spin>
  </section>
</template>

<script setup>
import { computed, h, onMounted, ref } from "vue";

const BriefSection = (props) =>
  h("div", { class: "brief-section" }, [
    h("div", { class: "section-title" }, [h("h3", props.title)]),
    ...(props.items?.length
      ? props.items.slice(0, 8).map((item) =>
          h("article", { class: "brief-item", key: item.url }, [
            h("a", { href: item.url, target: "_blank", rel: "noreferrer" }, item.title),
            h("div", { class: "brief-source" }, `${item.source || "未知来源"} · 重要度 ${item.importance ?? "-"}`),
            h("p", item.summary),
          ])
        )
      : [h("p", { class: "empty" }, "暂无内容")]),
  ]);

const report = ref(null);
const loading = ref(false);
const error = ref("");
const reportDate = ref("");
const briefUrl = (
  import.meta.env.VITE_DAILY_BRIEF_URL ||
  "https://raw.githubusercontent.com/edcv593/DailyBrief/gh-pages"
).replace(/\/$/, "");

const reportDateLabel = computed(() => (reportDate.value ? `更新于 ${reportDate.value}` : "最新日报"));

const formatChange = (value) => {
  if (value === undefined || value === null || value === "") return "暂无涨跌";
  const number = Number(value);
  return `${number >= 0 ? "+" : ""}${number.toFixed(2)}%`;
};

const loadBrief = async () => {
  loading.value = true;
  error.value = "";
  try {
    const response = await fetch(`${briefUrl}/index.json?ts=${Date.now()}`);
    if (!response.ok) throw new Error(`日报服务返回 ${response.status}`);
    report.value = await response.json();
    reportDate.value = report.value.date || "";
  } catch (err) {
    report.value = null;
    error.value = "暂时无法获取每日简报，请确认 DailyBrief 的 gh-pages 已生成，或配置 VITE_DAILY_BRIEF_URL。";
  } finally {
    loading.value = false;
  }
};

onMounted(loadBrief);
</script>

<style lang="scss" scoped>
.brief-page { padding: 16px 0 40px; }
.brief-heading, .section-title { display: flex; align-items: center; justify-content: space-between; gap: 16px; }
.brief-heading { margin-bottom: 24px; }
h1 { margin: 10px 0 6px; font-size: 34px; }
h2 { margin: 12px 0; font-size: clamp(24px, 4vw, 42px); line-height: 1.2; }
h3 { margin: 0; font-size: 20px; }
p { color: var(--n-text-color-2); line-height: 1.8; }
.brief-alert { margin-bottom: 20px; }
.hero-card, .brief-section, .trading-card, .editor-note { margin-bottom: 18px; border-radius: 18px; }
.hero-card { background: linear-gradient(135deg, var(--n-color), rgba(234, 68, 77, .1)); }
.report-meta, .brief-source, .section-title span { color: var(--n-text-color-3); font-size: 13px; }
.keywords { display: flex; flex-wrap: wrap; gap: 8px; margin-top: 18px; }
.brief-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 18px; }
.brief-section { padding: 22px; background: var(--n-color); }
.brief-item { padding: 16px 0; border-bottom: 1px solid var(--n-divider-color); }
.brief-item:last-child { border-bottom: 0; }
.brief-item a { color: var(--n-text-color-1); font-weight: 600; line-height: 1.5; text-decoration: none; }
.brief-item a:hover { color: var(--n-primary-color); }
.brief-item p { margin-top: 6px; font-size: 14px; }
.brief-source { margin-top: 7px; }
.empty { padding: 30px 0; text-align: center; }
.ticker-list { display: grid; grid-template-columns: repeat(4, 1fr); gap: 10px; margin-top: 18px; }
.ticker { display: flex; align-items: center; gap: 8px; padding: 12px; border-radius: 10px; background: var(--n-color-modal); }
.ticker span { flex: 1; overflow: hidden; color: var(--n-text-color-3); font-size: 12px; text-overflow: ellipsis; white-space: nowrap; }
.editor-note { margin-top: 18px; }
.editor-note h3 { margin-bottom: 10px; }
@media (max-width: 900px) { .brief-grid { grid-template-columns: 1fr; } .ticker-list { grid-template-columns: repeat(2, 1fr); } }
@media (max-width: 520px) { .brief-heading { align-items: flex-start; flex-direction: column; } h1 { font-size: 28px; } .ticker-list { grid-template-columns: 1fr; } }
</style>
