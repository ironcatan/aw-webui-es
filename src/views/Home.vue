<template lang="pug">
div
  h3 {{ $t('home.greeting') }}
  p {{ $t('home.intro') }}
  p(v-if="info && !info.version.includes('rust')")
    a(:href="apiBrowserUrl") {{ $t('home.apiBrowser') }}
  p
    small
      i
        | {{ $t('home.landingHint') }}
        |  #[router-link(to="/settings") {{ $t('nav.settings') }}].

</template>

<script lang="ts">
import { mapState } from 'pinia';
import { useServerStore } from '~/stores/server';

export default {
  name: 'Home',
  computed: {
    ...mapState(useServerStore, ['info']),
    apiBrowserUrl(): string {
      const base = window.location.pathname.replace(/[^/]*$/, '');
      return base + 'api/';
    },
  },
};
</script>
