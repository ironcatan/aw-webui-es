<template lang="pug">
div
  h3.mb-3 {{ $t('aiSummaryView.title') }}

  b-alert(variant="info" show)
    | {{ $t('aiSummaryView.disclaimerPrefix') }}

  div.row.mb-3
    div.col-md-4
      b-form-group(:label="$t('aiSummaryView.hostLabel')" label-class="font-weight-bold")
        b-form-select(v-model="selectedHost" :options="hostOptions")

    div.col-md-4
      b-form-group(:label="$t('workReport.dateRangeLabel')" label-class="font-weight-bold")
        b-form-select(v-model="dateRange" :options="dateRangeOptions")

    div.col-md-4
      b-form-group(:label="$t('aiSummaryView.providerLabel')" label-class="font-weight-bold")
        b-form-select(v-model="provider" :options="providerOptions" @change="onProviderChange")

  div.row.mb-3
    div.col-md-6
      b-form-group(:label="$t('aiSummaryView.apiKeyLabel')" label-class="font-weight-bold")
        b-form-input(
          v-model="apiKey"
          type="password"
          placeholder="sk-..."
          autocomplete="off"
          @blur="persistConfig"
        )

    div.col-md-6
      b-form-group(:label="$t('aiSummaryView.modelLabel')" label-class="font-weight-bold")
        b-form-input(v-model="model" :placeholder="$t('aiSummaryView.modelPlaceholder')" @blur="persistConfig")

  div.mb-3
    b-form-group(:label="$t('aiSummaryView.promptLabel')" label-class="font-weight-bold")
      b-form-textarea(v-model="userPrompt" rows="3" max-rows="8")

  div.mb-4
    b-button(@click="generate" variant="primary" :disabled="loading || !apiKey || !selectedHost")
      b-spinner.mr-2(v-if="loading" small)
      | {{ loading ? $t('aiSummaryView.generating') : $t('aiSummaryView.generateBtn') }}
    b-button.ml-2(
      v-if="aggregatedText"
      variant="outline-secondary"
      @click="dataVisible = !dataVisible"
    ) {{ dataVisible ? $t('aiSummaryView.hideRawData') : $t('aiSummaryView.showRawData') }}

  b-alert(v-if="error" variant="danger" show dismissible @dismissed="error = ''")
    | {{ error }}

  div(v-if="dataVisible && aggregatedText")
    b-card.mb-3
      template(slot="header")
        strong {{ $t('aiSummaryView.rawDataHeader') }}
      pre.mb-0(style="white-space: pre-wrap; font-size: 0.85em") {{ aggregatedText }}

  div(v-if="llmResponse")
    b-card
      template(slot="header")
        div.d-flex.justify-content-between.align-items-center
          strong {{ $t('nav.aiSummary') }}
          b-button(size="sm" variant="outline-secondary" @click="copyResponse")
            icon(name="copy")
            |  {{ copied ? $t('aiSummaryView.copied') : $t('aiSummaryView.copyBtn') }}
      div(style="white-space: pre-wrap") {{ llmResponse }}
</template>

<script lang="ts">
import { useBucketsStore } from '~/stores/buckets';
import { getClient } from '~/util/awclient';
import {
  aggregateEvents,
  buildSummaryText,
  loadLLMConfig,
  saveLLMConfig,
  callLLM,
  type LLMProvider,
} from '~/util/aiSummary';
import 'vue-awesome/icons/copy';

const DEFAULT_PROMPT =
  'Based on the following activity data, provide a concise summary of how I spent my time. ' +
  'Highlight the main activities, identify focus areas, and suggest any observations about productivity patterns.';

export default {
  name: 'AISummaryView',
  data() {
    const saved = loadLLMConfig();
    return {
      bucketsStore: useBucketsStore(),

      selectedHost: '',
      dateRange: 'last7d',
      provider: (saved.provider || 'openai') as LLMProvider,
      apiKey: saved.apiKey || '',
      model: saved.model || '',
      userPrompt: DEFAULT_PROMPT,

      loading: false,
      error: '',
      aggregatedText: '',
      llmResponse: '',
      dataVisible: false,
      copied: false,
    };
  },
  computed: {
    hostOptions(): { value: string; text: string }[] {
      const buckets = this.bucketsStore.buckets || [];
      const hosts = new Set<string>();
      for (const b of buckets) {
        if (b.type === 'currentwindow') {
          hosts.add(b.hostname);
        }
      }
      return Array.from(hosts).map(h => ({ value: h, text: h }));
    },
    dateRangeOptions() {
      return [
        { value: 'last7d', text: this.$t('workReport.last7dOption') },
        { value: 'last30d', text: this.$t('workReport.last30dOption') },
        { value: 'last90d', text: this.$t('aiSummaryView.last90dOption') },
      ];
    },
    providerOptions() {
      return [
        { value: 'openai', text: 'OpenAI' },
        { value: 'anthropic', text: 'Anthropic' },
      ];
    },
    periodDays(): number {
      return this.dateRange === 'last7d' ? 7 : this.dateRange === 'last30d' ? 30 : 90;
    },
    defaultModel(): string {
      if (this.provider === 'anthropic') return 'claude-haiku-4-5-20251001';
      return 'gpt-4o-mini';
    },
  },
  async mounted() {
    await this.bucketsStore.ensureLoaded();
    if (this.hostOptions.length > 0 && !this.selectedHost) {
      this.selectedHost = this.hostOptions[0].value;
    }
    if (!this.model) {
      this.model = this.defaultModel;
    }
  },
  methods: {
    onProviderChange() {
      if (
        !this.model ||
        this.model === 'gpt-4o-mini' ||
        this.model === 'claude-haiku-4-5-20251001'
      ) {
        this.model = this.defaultModel;
      }
      this.persistConfig();
    },
    persistConfig() {
      saveLLMConfig({
        provider: this.provider,
        apiKey: this.apiKey,
        model: this.model,
      });
    },
    async generate() {
      this.error = '';
      this.llmResponse = '';
      this.aggregatedText = '';

      if (!this.selectedHost) {
        this.error = this.$t('aiSummaryView.noHostError');
        return;
      }
      if (!this.apiKey.trim()) {
        this.error = this.$t('aiSummaryView.noApiKeyError');
        return;
      }

      this.loading = true;
      try {
        const events = await this.fetchEvents();
        const topApps = aggregateEvents(events);
        const totalDuration = topApps.reduce((s, a) => s + a.duration, 0);
        const summaryData = { topApps, totalDuration, periodDays: this.periodDays };
        const summaryText = buildSummaryText(summaryData);

        this.aggregatedText = summaryText;

        const fullPrompt = `${this.userPrompt}\n\n${summaryText}`;
        const effectiveModel = this.model.trim() || this.defaultModel;

        const response = await callLLM(
          {
            provider: this.provider,
            apiKey: this.apiKey.trim(),
            model: effectiveModel,
          },
          fullPrompt
        );
        this.llmResponse = response;
      } catch (err: any) {
        this.error = err?.message || String(err);
      } finally {
        this.loading = false;
      }
    },
    async fetchEvents(): Promise<any[]> {
      const buckets = this.bucketsStore.buckets || [];
      const windowBucket = buckets.find(
        b => b.type === 'currentwindow' && b.hostname === this.selectedHost
      );
      if (!windowBucket) {
        throw new Error(
          this.$t('aiSummaryView.noWindowBucketError', { host: this.selectedHost }) as string
        );
      }

      const end = new Date();
      const start = new Date(end.getTime() - this.periodDays * 24 * 60 * 60 * 1000);

      return getClient().getEvents(windowBucket.id, { start, end, limit: -1 });
    },
    async copyResponse() {
      try {
        await navigator.clipboard.writeText(this.llmResponse);
        this.copied = true;
        setTimeout(() => {
          this.copied = false;
        }, 2000);
      } catch {
        this.error = this.$t('aiSummaryView.copyError');
      }
    },
  },
};
</script>
