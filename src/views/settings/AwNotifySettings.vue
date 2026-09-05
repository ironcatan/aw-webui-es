<template lang="pug">
div
  div.d-flex.justify-content-between.align-items-center.mb-3
    div
      h5.mb-1 {{ $t('awNotify.title') }}
      small.text-muted {{ $t('awNotify.subtitle') }}
    b-btn(@click="save" size="sm" variant="primary" :disabled="saving || loading")
      | {{ saving ? $t('awNotify.saving') : $t('common.save') }}

  b-alert(v-if="error" show variant="danger") {{ error }}
  b-alert(v-if="success" show variant="success" dismissible @dismissed="success = false") {{ $t('awNotify.settingsSaved') }}

  div(v-if="loading")
    b-spinner(small) {{ $t('common.loading') }}

  div(v-else)
    p.text-muted.small.mb-3
      | {{ $t('awNotify.description') }}

    div(v-if="alerts.length === 0")
      p.text-muted.font-italic {{ $t('awNotify.noAlerts') }}

    b-card.mb-2(v-for="(alert, idx) in alerts" :key="idx")
      div.d-flex.align-items-start
        div.flex-grow-1
          b-form-group(:label="$t('awNotify.labelField')" label-cols-sm="3" label-size="sm")
            b-input(v-model="alert.label" size="sm" :placeholder="$t('awNotify.labelPlaceholder')")
          b-form-group(:label="$t('alerts.categoryLabel')" label-cols-sm="3" label-size="sm")
            b-input(
              v-model="alert.category"
              size="sm"
              :placeholder="$t('awNotify.categoryPlaceholder')"
            )
            small.form-text.text-muted
              | {{ $t('awNotify.categoryHelp') }}
          b-form-group(
            :label="$t('awNotify.thresholdsField')"
            label-cols-sm="3"
            label-size="sm"
            :invalid-feedback="thresholdError(alert.thresholdStr)"
            :state="thresholdState(alert.thresholdStr)"
          )
            b-input(
              v-model="alert.thresholdStr"
              size="sm"
              :placeholder="$t('awNotify.thresholdsPlaceholder')"
              :state="thresholdState(alert.thresholdStr)"
            )
            small.form-text.text-muted {{ $t('awNotify.thresholdsHelp') }}
          b-form-group(:label="$t('awNotify.typeField')" label-cols-sm="3" label-size="sm")
            b-form-radio-group(v-model="alert.positive" :options="goalOptions" size="sm")
        b-btn.ml-2(@click="removeAlert(idx)" variant="outline-danger" size="sm" :title="$t('awNotify.removeAlertTitle')")
          icon(name="trash")

    b-btn.mt-1(@click="addAlert" variant="outline-secondary" size="sm")
      icon(name="plus")
      |  {{ $t('alerts.addAlertBtn') }}
</template>

<script lang="ts">
import 'vue-awesome/icons/plus';
import 'vue-awesome/icons/trash';

import { getClient } from '~/util/awclient';
import {
  AwNotifyAlert,
  AwNotifyConfig,
  parseAwNotifyConfig,
  parseThresholds,
} from '~/util/aw-notify';

const SETTINGS_KEY = 'aw-notify';

interface AlertRow {
  label: string;
  category: string;
  thresholdStr: string;
  positive: boolean;
}

function dtoToRow(dto: AwNotifyAlert): AlertRow {
  return {
    label: dto.label ?? '',
    category: dto.category,
    thresholdStr: dto.thresholds_minutes.join(', '),
    positive: dto.positive,
  };
}

function rowToDto(row: AlertRow): AwNotifyAlert {
  const thresholds = parseThresholds(row.thresholdStr);
  if (!thresholds) {
    throw new Error('Thresholds must be comma-separated positive whole minutes.');
  }
  return {
    label: row.label.trim() || null,
    category: row.category.trim() || 'All',
    thresholds_minutes: thresholds,
    positive: row.positive,
  };
}

export default {
  name: 'AwNotifySettings',
  data() {
    return {
      alerts: [] as AlertRow[],
      config: {} as AwNotifyConfig,
      loading: false,
      saving: false,
      error: '',
      success: false,
      goalOptions: [],
    };
  },
  async mounted() {
    this.goalOptions = [
      { text: this.$t('awNotify.warningOption'), value: false },
      { text: this.$t('awNotify.goalOption'), value: true },
    ];
    await this.load();
  },
  methods: {
    async load() {
      this.loading = true;
      this.error = '';
      try {
        const client = getClient();
        const resp = await client.req.get(`/0/settings/${SETTINGS_KEY}`);
        // The setting hasn't been saved yet: server returns 200 with a null
        // body (not a 404), so this must be treated the same as "not found"
        // rather than as a malformed/corrupt config.
        if (resp.data === null) {
          this.config = {} as AwNotifyConfig;
          this.alerts = this.defaultAlerts();
          return;
        }
        const config = parseAwNotifyConfig(resp.data);
        if (!config) {
          throw new Error(this.$t('awNotify.unsupportedFormat') as string);
        }
        this.config = config;
        this.alerts = config.alerts.map(dtoToRow);
      } catch (e: any) {
        if (e?.response?.status === 404) {
          this.config = {} as AwNotifyConfig;
          this.alerts = this.defaultAlerts();
        } else {
          this.error = this.$t('awNotify.loadFailedPrefix', { message: e?.message ?? e });
        }
      } finally {
        this.loading = false;
      }
    },
    async save() {
      this.error = '';
      this.success = false;
      if (this.alerts.some(row => parseThresholds(row.thresholdStr) === null)) {
        this.error = this.$t('awNotify.thresholdsInvalid');
        return;
      }
      this.saving = true;
      try {
        const client = getClient();
        const payload: AwNotifyConfig = { ...this.config, alerts: this.alerts.map(rowToDto) };
        await client.req.post(`/0/settings/${SETTINGS_KEY}`, payload, {
          headers: { 'Content-Type': 'application/json' },
        });
        this.success = true;
      } catch (e: any) {
        this.error = this.$t('awNotify.saveFailedPrefix', { message: e?.message ?? e });
      } finally {
        this.saving = false;
      }
    },
    thresholdError(value: string): string {
      return parseThresholds(value) === null ? this.$t('awNotify.useCommaSeparated') : '';
    },
    thresholdState(value: string): boolean | null {
      return parseThresholds(value) === null ? false : null;
    },
    addAlert() {
      this.alerts.push({
        label: '',
        category: 'All',
        thresholdStr: '60, 120',
        positive: false,
      });
    },
    removeAlert(idx: number) {
      this.alerts.splice(idx, 1);
    },
    defaultAlerts(): AlertRow[] {
      return [
        { label: 'All', category: 'All', thresholdStr: '60, 240, 480', positive: false },
        { label: '💼 Work', category: 'Work', thresholdStr: '60, 120, 240', positive: true },
      ];
    },
  },
};
</script>
