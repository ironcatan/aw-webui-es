<template lang="pug">
div
  h3 {{ $t('devView.title') }}
  | {{ $t('devView.intro') }}

  p
    | {{ $t('devView.empty') }}
</template>

<script lang="ts">
export default {
  name: 'aw-dev',
};
</script>
