<script setup lang="ts">
import { inBrowser } from 'vitepress'
import { computed, provide, watchEffect } from 'vue'

import { useData } from '../composables/data'
import { navInjectionKey, useNav } from '../composables/nav'
import VPComiCineNav from './VPComiCineNav.vue'

const { closeScreen } = useNav()
const { frontmatter } = useData()

const hasNavbar = computed(() => {
  return frontmatter.value.navbar !== false
})

provide(navInjectionKey, { closeScreen })

watchEffect(() => {
  if (inBrowser) {
    document.documentElement.classList.toggle('hide-nav', !hasNavbar.value)
  }
})
</script>

<template>
  <header v-if="hasNavbar" class="VPNav">
    <!--
      ComiCine fork：页头改为与主站 base.html 的 nav.site-nav 一致的结构。
      原来的 VPNavBar / VPNavScreen 不再渲染；移动端菜单由 VPLocalNav 提供。
    -->
    <VPComiCineNav />
  </header>
</template>

<style scoped>
.VPNav {
  position: relative;
  top: var(--vp-layout-top-height, 0px);
  inset-inline-start: 0;
  z-index: var(--vp-z-index-nav);
  width: 100%;
  pointer-events: none;
}

@media (min-width: 60rem) {
  .VPNav {
    position: fixed;
  }
}
</style>
