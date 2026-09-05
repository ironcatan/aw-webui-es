<template lang="pug">
div.container(style="color: #555; font-size: 0.9em")
  div.mb-2
    | {{ $t('footer.basedOn') }}
    | &nbsp;·&nbsp;{{ $t('footer.by') }}&nbsp;·&nbsp;
    a(href="https://github.com/ironcatan/chronly-docker", target="_blank", rel="noopener noreferrer") {{ $t('footer.github') }}
    div
      span.mt-2(v-if="info", style="color: #888; font-size: 0.8em")
        span.mr-2
          b {{ $t('footer.host') }}
          | &nbsp; {{info.hostname}}
        span
          b {{ $t('footer.version') }}
          | &nbsp; {{info.version}}
</template>

<script lang="ts">
import { mapState } from 'pinia';
import { useServerStore } from '~/stores/server';

export default {
  name: 'Footer',
  computed: {
    ...mapState(useServerStore, ['info']),
  },
};
</script>
