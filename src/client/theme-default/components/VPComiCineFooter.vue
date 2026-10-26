<script setup lang="ts">
/**
 * ComiCine footer — structure and styling match the main site footer.
 *
 * 这是 fork 定制件：不使用 VitePress 自带的 VPFooter。
 * The main site is fixed-height, but the docs page scrolls and keeps its footer in normal flow.
 * Typography and separators follow the main site.
 */
import { computed, onMounted, ref } from 'vue'
import { useData } from 'vitepress'

import { useLayout } from '../composables/layout'

const { theme, frontmatter } = useData()
const { hasSidebar } = useLayout()

interface FooterItem {
  text: string
  link?: string
}

const footer = computed<any>(() => (theme.value as any).footer ?? {})
const items = computed<FooterItem[]>(() => footer.value.items ?? [])
const copyrightYear = ref(new Date().getUTCFullYear())
onMounted(() => {
  try {
    const timeZone = Intl.DateTimeFormat().resolvedOptions().timeZone
    const yearText = timeZone
      ? new Intl.DateTimeFormat('en-US', { timeZone, year: 'numeric' }).format(new Date())
      : ''
    const year = Number(yearText)
    copyrightYear.value = Number.isInteger(year) ? year : new Date().getUTCFullYear()
  } catch {
    copyrightYear.value = new Date().getUTCFullYear()
  }
})
const copyright = computed(() => {
  const markup = footer.value.copyright
  if (!markup) return ''
  return markup.replace(/©\s*(\d{4})(?:-\d{4})?/, (_match: string, startYearText: string) => {
    const startYear = Number(startYearText)
    return `© ${startYear === copyrightYear.value ? startYear : `${startYear}-${copyrightYear.value}`}`
  })
})
const show = computed(
  () => frontmatter.value.footer !== false && !!(footer.value.copyright || items.value.length)
)
</script>

<template>
  <footer
    v-if="show"
    class="VPComiCineFooter"
    :class="{ 'has-sidebar': hasSidebar }"
    id="global-footer"
  >
    <div class="footer-link">
      <div class="footer-text">
        <span
          v-if="copyright"
          class="footer-item footer-copyright"
          v-html="copyright"
        />
        <span
          v-for="(item, index) in items"
          :key="index"
          class="footer-item"
        >
          <a
            v-if="item.link"
            class="footer-source-link"
            :href="item.link"
            target="_blank"
            rel="noopener noreferrer"
          >{{ item.text }}</a>
          <template v-else>{{ item.text }}</template>
        </span>
      </div>
    </div>
  </footer>
</template>
