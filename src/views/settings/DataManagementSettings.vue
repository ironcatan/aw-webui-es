<template lang="pug">
div
  div.d-flex.align-items-center.mb-4
    span.text-muted
      | {{ $t('settings.dataManagement.dbSizeLabel') }}
      strong.ml-1 {{ storageSizeText }}
    b-btn.ml-2(@click="fetchStorageSize" variant="outline-secondary" size="sm" :disabled="storageSizeLoading")
      icon(name="sync")

  h5.mb-2 {{ $t('settings.dataManagement.backupTitle') }}
  p.text-muted.small {{ $t('settings.dataManagement.backupHint') }}
  b-btn(@click="downloadBackup" variant="outline-primary" size="sm" :disabled="backupInProgress")
    icon.mr-1(name="download")
    | {{ $t('settings.dataManagement.downloadBtn') }}
  b-alert.mt-2(:show="!!backupError" variant="danger" dismissible @dismissed="backupError = ''")
    | {{ backupError }}

  hr.my-4

  h5.mb-2 {{ $t('settings.dataManagement.restoreTitle') }}
  p.text-muted.small {{ $t('settings.dataManagement.restoreHint') }}
  label.btn.btn-sm.btn-outline-secondary(style="margin: 0")
    | {{ $t('settings.dataManagement.restoreBtn') }}
    input(type="file" accept=".json,application/json" @change="onRestoreFileSelected" hidden)
  b-alert.mt-2(:show="!!restoreMessage" :variant="restoreSuccess ? 'success' : 'danger'" dismissible @dismissed="restoreMessage = ''")
    | {{ restoreMessage }}

  hr.my-4

  h5.mb-2 {{ $t('settings.dataManagement.purgeTitle') }}
  p.text-muted.small {{ $t('settings.dataManagement.purgeHint') }}
  b-form-group(:label="$t('settings.dataManagement.purgeDateLabel')" label-cols-md=4)
    b-form-input(v-model="purgeDate" type="date" :max="todayStr")
  b-btn(@click="openPurgeModal" variant="outline-danger" size="sm" :disabled="!purgeDate")
    icon.mr-1(name="trash")
    | {{ $t('settings.dataManagement.purgeBtn') }}
  b-alert.mt-2(:show="!!purgeMessage" :variant="purgeSuccess ? 'success' : 'danger'" dismissible @dismissed="purgeMessage = ''")
    | {{ purgeMessage }}

  b-modal(
    v-model="showPurgeModal"
    :title="$t('settings.dataManagement.purgeModalTitle')"
    :ok-title="$t('settings.dataManagement.purgeModalConfirmBtn')"
    ok-variant="danger"
    :cancel-title="$t('common.cancel')"
    @ok="confirmPurge"
  )
    p {{ $t('settings.dataManagement.purgeModalBody', { date: purgeDate }) }}
</template>

<script lang="ts">
import 'vue-awesome/icons/download';
import 'vue-awesome/icons/trash';
import 'vue-awesome/icons/sync';
import { getClient } from '~/util/awclient';
import { downloadFile } from '~/util/export';
import { shouldAttemptJsonImport } from '~/util/importFile';

function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  const units = ['KB', 'MB', 'GB', 'TB'];
  let value = bytes;
  let unitIndex = -1;
  do {
    value /= 1024;
    unitIndex++;
  } while (value >= 1024 && unitIndex < units.length - 1);
  return `${value.toFixed(1)} ${units[unitIndex]}`;
}

export default {
  name: 'DataManagementSettings',
  data() {
    return {
      storageSizeBytes: null as number | null,
      storageSizeLoading: false,
      backupInProgress: false,
      backupError: '',
      restoreMessage: '',
      restoreSuccess: false,
      purgeDate: '',
      showPurgeModal: false,
      purgeMessage: '',
      purgeSuccess: false,
    };
  },
  computed: {
    todayStr(): string {
      return new Date().toISOString().slice(0, 10);
    },
    storageSizeText(): string {
      if (this.storageSizeLoading) return this.$t('common.loading') as string;
      if (this.storageSizeBytes === null) return '—';
      return formatBytes(this.storageSizeBytes);
    },
  },
  created() {
    this.fetchStorageSize();
  },
  methods: {
    async fetchStorageSize() {
      this.storageSizeLoading = true;
      try {
        const client = getClient();
        const resp = await client.req.get('/0/storage_size');
        this.storageSizeBytes = resp.data.size;
      } catch (e) {
        console.error('Failed to fetch storage size', e);
        this.storageSizeBytes = null;
      } finally {
        this.storageSizeLoading = false;
      }
    },
    async downloadBackup() {
      this.backupInProgress = true;
      this.backupError = '';
      try {
        const client = getClient();
        const [exportResp, settingsResp] = await Promise.all([
          client.req.get('/0/export'),
          client.req.get('/0/settings'),
        ]);
        const payload = { ...exportResp.data, settings: settingsResp.data };
        const text = JSON.stringify(payload, null, 2);
        const date = new Date().toISOString().slice(0, 10);
        await downloadFile(`aw-backup-${date}.json`, text, 'application/json');
      } catch (e) {
        console.error('Backup export failed', e);
        this.backupError = this.$t('settings.dataManagement.backupError') as string;
      } finally {
        this.backupInProgress = false;
      }
    },
    async onRestoreFileSelected(elem) {
      const file = elem.target.files[0];
      if (!file) return;
      elem.target.value = '';

      if (!shouldAttemptJsonImport(file)) {
        this.restoreSuccess = false;
        this.restoreMessage = this.$t('settings.dataManagement.restoreWrongType') as string;
        return;
      }

      if (!confirm(this.$t('settings.dataManagement.restoreConfirmPrompt') as string)) {
        return;
      }

      try {
        const parsed = JSON.parse(await file.text());
        const client = getClient();
        await client.req.post('/0/import', parsed);
        if (parsed.settings && typeof parsed.settings === 'object') {
          for (const [key, value] of Object.entries(parsed.settings)) {
            await client.req.post(`/0/settings/${key}`, value);
          }
        }
        this.restoreSuccess = true;
        this.restoreMessage = this.$t('settings.dataManagement.restoreSuccess') as string;
      } catch (e) {
        console.error('Restore failed', e);
        this.restoreSuccess = false;
        this.restoreMessage = this.$t('settings.dataManagement.restoreError') as string;
      }
    },
    openPurgeModal() {
      if (!this.purgeDate) return;
      this.showPurgeModal = true;
    },
    async confirmPurge() {
      try {
        const client = getClient();
        const end = new Date(this.purgeDate + 'T00:00:00').toISOString();
        const resp = await client.req.post('/0/purge', { end });
        const total = Object.values(resp.data as Record<string, number>).reduce(
          (a: number, b: number) => a + b,
          0
        );
        this.purgeSuccess = true;
        this.purgeMessage = this.$t('settings.dataManagement.purgeSuccess', {
          count: total,
        }) as string;
        this.fetchStorageSize();
      } catch (e) {
        console.error('Purge failed', e);
        this.purgeSuccess = false;
        this.purgeMessage = this.$t('settings.dataManagement.purgeError') as string;
      }
    },
  },
};
</script>
