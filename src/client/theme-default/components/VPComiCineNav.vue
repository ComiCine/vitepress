<script setup lang="ts">
/**
 * ComiCine header — structure and styling match the main site navigation.
 *
 * This fork component replaces VitePress default navigation.
 * The brand name uses the same text wordmark as the main site.
 */
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import { useData, useRoute, useRouter } from 'vitepress'

import { useNav } from '../composables/nav'
import VPIcon from './VPIcon.vue'

interface LangItem {
  code: string
  label: string
  flag?: string
  default?: boolean
}

const { theme, isDark } = useData()
const route = useRoute()
const router = useRouter()
const {
  isScreenOpen: navMenuOpen,
  screenTriggerEl,
  toggleScreen,
  closeScreen
} = useNav()

const comicine = computed<any>(() => (theme.value as any).comicine ?? {})
const languages = computed<LangItem[]>(() => comicine.value.languages ?? [])
const defaultLang = computed(
  () =>
    languages.value.find((l) => l.default)?.code ??
    languages.value[0]?.code ??
    ''
)

const langCodes = computed(() => languages.value.map((l) => l.code))

/**
 * themeConfig.comicine.langSegmentAlways：内容页路径是否**总是**带语言段
 * （含默认语言，如 /video-platform/zh-cn）。默认 false = 旧行为（默认语言无后缀）。
 * 首页永远是例外：`/` 就是默认语言。
 */
const langSegmentAlways = computed(
  () => (comicine.value as any).langSegmentAlways === true
)

/** 语言码是路径的最后一段：/video-platform/en-us；默认语言无后缀。 */
function langOfPath(path: string): string {
  const pathname = path.split('?', 1)[0] ?? ''
  const segments = pathname.split('/').filter(Boolean)
  const last = segments.at(-1)
  return last && langCodes.value.includes(last) ? last : defaultLang.value
}

const currentLang = computed(() => langOfPath(route.path))

function pathForLang(code: string): string {
  const pathname = route.path.split('?', 1)[0] ?? ''
  const segments = pathname.split('/').filter(Boolean)
  const last = segments.at(-1)
  if (last && langCodes.value.includes(last)) {
    segments.pop()
  }
  const base = segments.length ? '/' + segments.join('/') : '/'
  // 首页：`/` 是默认语言，其它语言是 /en-us、/ja-jp…
  if (base === '/') return code === defaultLang.value ? '/' : `/${code}`
  // 内容页：默认语言要不要后缀由 langSegmentAlways 决定
  if (code === defaultLang.value && !langSegmentAlways.value) return base
  return `${base}/${code}`
}

const brandLink = computed(() => {
  const code = currentLang.value
  return code === defaultLang.value ? '/' : `/${code}`
})

interface NavItem {
  text: string
  link: string
  activeMatch?: string
}

const navItems = computed<NavItem[]>(() => {
  const nav = (theme.value as any).nav
  if (!Array.isArray(nav)) return []
  return nav
    .filter((item: any) => item && item.link)
    .map((item: any) => ({
      text: item.text,
      link: item.link,
      activeMatch: item.activeMatch
    }))
})

function isActive(item: NavItem): boolean {
  const path = route.path
  const link = item.link.split('?', 1)[0] ?? item.link
  if (item.activeMatch) return new RegExp(item.activeMatch).test(path)
  if (/^https?:/.test(link)) return false
  if (link === '/') return path === '/' || path === `/${currentLang.value}`
  return path === link || path.startsWith(link + '/')
}

function isExternal(link: string): boolean {
  return /^https?:\/\//.test(link)
}

const socialLinks = computed<any[]>(() => (theme.value as any).socialLinks ?? [])

const menuOpen = ref(false)
const langRoot = ref<HTMLElement | null>(null)
const navMenuButton = ref<HTMLButtonElement | null>(null)
const langAccordionOpen = ref(false)

function toggleAppearance() {
  isDark.value = !isDark.value
}

/* 桌面端悬停展开/收起（与主站 base.js 的 bindLangHover 一致），
   触摸端保持点击切换。 */
const HOVER_QUERY = '(hover: hover) and (pointer: fine)'

function canHover(): boolean {
  return (
    typeof window !== 'undefined' && window.matchMedia(HOVER_QUERY).matches
  )
}

function onLangEnter() {
  if (canHover()) menuOpen.value = true
}

function onLangLeave() {
  if (canHover()) menuOpen.value = false
}

function toggleMenu() {
  menuOpen.value = !menuOpen.value
  navMenuOpen.value = false
}

function closeMenu() {
  menuOpen.value = false
}

function toggleNavMenu() {
  menuOpen.value = false
  langAccordionOpen.value = false
  toggleScreen()
}

function closeNavMenu() {
  closeScreen()
  langAccordionOpen.value = false
}

function onDocumentClick(event: MouseEvent) {
  if (menuOpen.value && langRoot.value && !langRoot.value.contains(event.target as Node)) {
    closeMenu()
  }
}

function onDocumentKeydown(event: KeyboardEvent) {
  if (event.key !== 'Escape') return
  if (navMenuOpen.value) {
    event.preventDefault()
    closeNavMenu()
    navMenuButton.value?.focus()
  } else if (menuOpen.value) {
    closeMenu()
  }
}

function goLang(code: string) {
  closeMenu()
  closeNavMenu()
  const target = pathForLang(code)
  if (target !== route.path) router.go(target)
}

watch(navMenuOpen, (open) => {
  document.documentElement.classList.toggle('nav-screen-open', open)
})

onMounted(() => {
  document.documentElement.classList.toggle('nav-screen-open', navMenuOpen.value)
  screenTriggerEl.value = navMenuButton.value
  document.addEventListener('click', onDocumentClick)
  document.addEventListener('keydown', onDocumentKeydown)
})
onBeforeUnmount(() => {
  document.removeEventListener('click', onDocumentClick)
  document.removeEventListener('keydown', onDocumentKeydown)
  document.documentElement.classList.remove('nav-screen-open')
  if (screenTriggerEl.value === navMenuButton.value) screenTriggerEl.value = null
})
</script>

<template>
  <div class="VPComiCineNav">
    <div class="nav-inner">
      <a :href="brandLink" class="nav-brand-link" aria-label="ComiCine">
        <span class="nav-brand brand-word">ComiCine</span>
      </a>

      <div class="nav-right">
        <div v-if="navItems.length" class="global-nav">
          <a
            v-for="item in navItems"
            :key="item.link"
            :href="item.link"
            class="nav-link"
            :class="{ active: isActive(item) }"
          >
            <span>{{ item.text }}</span>
            <svg
              v-if="isExternal(item.link)"
              class="nav-external"
              viewBox="0 0 24 24"
              fill="currentColor"
              aria-hidden="true"
            >
              <path d="M0 0h24v24H0V0z" fill="none" />
              <path d="M9 5v2h6.59L4 18.59 5.41 20 17 8.41V15h2V5H9z" />
            </svg>
          </a>
        </div>
        <span class="nav-divider" />

        <div
          v-if="languages.length > 1"
          ref="langRoot"
          class="lang-toggle"
          @mouseenter="onLangEnter"
          @mouseleave="onLangLeave"
        >
          <button
            type="button"
            class="lang-btn"
            aria-haspopup="true"
            :aria-expanded="menuOpen"
            :aria-label="(theme as any).langMenuLabel || 'Language'"
            @click.stop="toggleMenu"
          >
            <svg
              class="lang-icon"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              aria-hidden="true"
            >
              <path
                d="m5 8 6 6M4 14l6-6 2-3M2 5h12M7 2h1M22 22l-5-10-5 10M14 18h6"
              />
            </svg>
            <svg
              class="lang-chevron"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              aria-hidden="true"
            >
              <path d="m6 9 6 6 6-6" />
            </svg>
          </button>
          <div class="lang-menu" :class="{ open: menuOpen }">
            <a
              v-for="lang in languages"
              :key="lang.code"
              class="lang-option"
              :class="{ selected: lang.code === currentLang }"
              :href="pathForLang(lang.code)"
              @click.prevent="goLang(lang.code)"
            >
              <span v-if="lang.flag" class="lang-flag">{{ lang.flag }}</span>
              <span>{{ lang.label }}</span>
            </a>
          </div>
        </div>
        <span class="nav-divider nav-divider-lang-theme" />

        <button
          type="button"
          class="appearance-switch"
          role="switch"
          :aria-checked="isDark ? 'true' : 'false'"
          :aria-label="(theme as any).darkModeSwitchLabel || 'Appearance'"
          @click="toggleAppearance"
        >
          <span class="appearance-check">
            <span class="appearance-icon">
              <svg
                class="sun"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <circle cx="12" cy="12" r="4" />
                <path
                  d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M6.34 17.66l-1.41 1.41M19.07 4.93l-1.41 1.41"
                />
              </svg>
              <svg
                class="moon"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                aria-hidden="true"
              >
                <path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z" />
              </svg>
            </span>
          </span>
        </button>
        <span class="nav-divider" />
        <div class="nav-mobile-menu">
          <button
            ref="navMenuButton"
            type="button"
            class="icon-btn nav-menu-button"
            :aria-expanded="navMenuOpen"
            aria-controls="nav-screen"
            :aria-label="(theme as any).navMenuLabel || 'Menu'"
            :class="{ active: navMenuOpen }"
            @click.stop="toggleNavMenu"
          >
            <span class="nav-menu-icon" aria-hidden="true">
              <span class="top" />
              <span class="middle" />
              <span class="bottom" />
            </span>
          </button>
        </div>

        <a
          v-for="(link, index) in socialLinks"
          :key="index"
          class="icon-btn nav-social-link"
          :href="link.link"
          target="_blank"
          rel="noopener noreferrer"
          :aria-label="link.ariaLabel || link.icon"
        >
          <VPIcon :icon="String(link.icon).includes(':') ? link.icon : `simple-icons:${link.icon}`" />
        </a>
      </div>
    </div>

    <div
      id="nav-screen"
      class="nav-screen"
      :class="{ open: navMenuOpen }"
      :aria-hidden="!navMenuOpen"
      :inert="!navMenuOpen"
      @click.self="closeNavMenu"
    >
      <div class="nav-screen-container">
        <nav class="nav-screen-links" :aria-label="(theme as any).navMenuLabel || 'Main Navigation'">
          <a
            v-for="item in navItems"
            :key="item.link"
            class="nav-screen-link"
            :class="{ active: isActive(item) }"
            :href="item.link"
            @click="closeNavMenu"
          >
            <span>{{ item.text }}</span>
            <svg
              v-if="isExternal(item.link)"
              class="nav-external"
              viewBox="0 0 24 24"
              fill="currentColor"
              aria-hidden="true"
            >
              <path d="M0 0h24v24H0V0z" fill="none" />
              <path d="M9 5v2h6.59L4 18.59 5.41 20 17 8.41V15h2V5H9z" />
            </svg>
          </a>
        </nav>

        <section v-if="languages.length > 1" class="nav-screen-languages">
          <button
            type="button"
            class="nav-screen-section-toggle"
            :aria-label="(theme as any).langMenuLabel || 'Language'"
            :aria-expanded="langAccordionOpen"
            aria-controls="nav-screen-language-list"
            @click="langAccordionOpen = !langAccordionOpen"
          >
            <svg class="nav-screen-language-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <path d="m5 8 6 6M4 14l6-6 2-3M2 5h12M7 2h1M22 22l-5-10-5 10M14 18h6" />
            </svg>
            <svg class="nav-screen-chevron" viewBox="0 0 24 24" fill="none" aria-hidden="true">
              <path d="m6 9 6 6 6-6" />
            </svg>
          </button>
          <ul
            id="nav-screen-language-list"
            class="nav-screen-language-list"
            :class="{ open: langAccordionOpen }"
            :aria-hidden="!langAccordionOpen"
            :inert="!langAccordionOpen"
          >
            <li v-for="lang in languages" :key="lang.code">
              <a
                class="nav-screen-language-link"
                :class="{ selected: lang.code === currentLang }"
                :href="pathForLang(lang.code)"
                @click.prevent="goLang(lang.code)"
              >
                <span v-if="lang.flag" class="lang-flag">{{ lang.flag }}</span>
                <span>{{ lang.label }}</span>
              </a>
            </li>
          </ul>
        </section>

        <div class="nav-screen-appearance">
          <span>{{ (theme as any).darkModeSwitchLabel || 'Appearance' }}</span>
          <button
            type="button"
            class="appearance-switch"
            role="switch"
            :aria-checked="isDark ? 'true' : 'false'"
            :aria-label="(theme as any).darkModeSwitchLabel || 'Appearance'"
            @click="toggleAppearance"
          >
            <span class="appearance-check">
              <span class="appearance-icon">
                <svg class="sun" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                  <circle cx="12" cy="12" r="4" />
                  <path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M6.34 17.66l-1.41 1.41M19.07 4.93l-1.41 1.41" />
                </svg>
                <svg class="moon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                  <path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z" />
                </svg>
              </span>
            </span>
          </button>
        </div>

        <div v-if="socialLinks.length" class="nav-screen-socials">
          <a
            v-for="(link, index) in socialLinks"
            :key="index"
            class="nav-screen-social-link"
            :href="link.link"
            target="_blank"
            rel="noopener noreferrer"
            :aria-label="link.ariaLabel || link.icon"
            @click="closeNavMenu"
          >
            <VPIcon :icon="String(link.icon).includes(':') ? link.icon : `simple-icons:${link.icon}`" />
          </a>
        </div>
      </div>
    </div>
  </div>
</template>
