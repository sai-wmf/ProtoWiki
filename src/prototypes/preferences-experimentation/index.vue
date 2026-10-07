<script setup lang="ts">
import ArticleLive from '@/components/article/ArticleLive.vue'

import ExperimentationChrome from './ExperimentationChrome.vue'

definePage({
  meta: {
    title: 'Experimentation preferences',
    description:
      'Prototyping a new preferences section to inform users and allow them to opt-out from experimentation.',
    category: 'prototype',
    platform: 'web',
  },
})
</script>

<template>
  <ExperimentationChrome open-notices>
    <ArticleLive article="Main Page" :blank-titlebar="true" />
  </ExperimentationChrome>
</template>
