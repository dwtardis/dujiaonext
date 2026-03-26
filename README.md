# HOME页增加公告栏
1.在 `\src\views\Home.vue` 修改如下

在第 2 行 `<div class="home-page min-h-screen theme-page">` 之后，第 5 行 `<template v-if="templateMode === 'list'">` 之前，插入公告横幅 HTML。

```vue
<!-- Notice Bar -->
<div
  v-if="latestNotice && !noticeBarDismissed"
  class="relative z-20 border-b theme-border bg-blue-50 dark:bg-blue-950/30"
>
  <div class="container mx-auto px-4">
    <div class="flex items-center justify-between gap-4 py-3">
      <div class="flex items-center gap-3 min-w-0">
        <span class="flex-shrink-0 flex items-center justify-center w-8 h-8 rounded-full bg-blue-100 dark:bg-blue-900/50 text-blue-600 dark:text-blue-400">
          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
              d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
          </svg>
        </span>
        <router-link
          :to="`/blog/${latestNotice.slug}`"
          class="text-sm font-medium text-blue-800 dark:text-blue-200 truncate hover:underline"
        >
          {{ getLocalizedText(latestNotice.title) }}
          <span v-if="getLocalizedText(latestNotice.summary)" class="ml-2 text-blue-600/70 dark:text-blue-300/70 font-normal">
            — {{ getLocalizedText(latestNotice.summary) }}
          </span>
        </router-link>
      </div>
      <button
        type="button"
        class="flex-shrink-0 p-1 rounded-md text-blue-500 hover:text-blue-700 dark:text-blue-400 dark:hover:text-blue-200 hover:bg-blue-100 dark:hover:bg-blue-900/50 transition-colors"
        @click="noticeBarDismissed = true"
      >
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
        </svg>
      </button>
    </div>
  </div>
</div>
```

2.在第 390 行（const quickBuyVisible = ref(false) 之后），插入：
```vue
const latestNotice = ref<any>(null)
const noticeBarDismissed = ref(false)

const loadLatestNotice = async () => {
  try {
    const response = await postAPI.list({ type: 'notice', page: 1, page_size: 1 })
    const notices = response.data.data || []
    if (notices.length > 0) {
      const notice = notices[0]
      const title = getLocalizedText(notice.title)
      if (title && title.trim()) {
        latestNotice.value = notice
      }
    }
  } catch (error) {
    console.error('Failed to load latest notice:', error)
  }
}
```

3.onMounted 中加入调用
```vue
onMounted(async () => {
  if (templateMode.value === 'list') {
    await Promise.all([loadBanners(), listInitialize()])
  } else {
    await Promise.all([loadBanners(), loadFeaturedProducts(), loadLatestPosts()])
  }
})
```
改为
```vue
onMounted(async () => {
  if (templateMode.value === 'list') {
    await Promise.all([loadBanners(), listInitialize(), loadLatestNotice()])
  } else {
    await Promise.all([loadBanners(), loadFeaturedProducts(), loadLatestPosts(), loadLatestNotice()])
  }
})
```

# Home.vue — 添加浮动 Telegram 图标
在模板最末尾、</div> 关闭标签之前（第 393 行 </ProductQuickBuy> 之后、第 394 行 </div> 之前）添加：
```vue
    <!-- Floating Telegram Button -->
    <a
      v-if="appStore.config?.contact?.telegram"
      :href="appStore.config.contact.telegram"
      target="_blank"
      rel="noopener noreferrer"
      class="fixed right-5 bottom-24 md:bottom-8 z-50 w-12 h-12 rounded-full bg-[#2AABEE] text-white shadow-lg hover:shadow-xl hover:scale-110 transition-all flex items-center justify-center"
    >
      <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
        <path d="M12 0C5.373 0 0 5.373 0 12s5.373 12 12 12 12-5.373 12-12S18.627 0 12 0zm5.894 8.221l-1.97 9.28c-.145.658-.537.818-1.084.508l-3-2.21-1.446 1.394c-.14.18-.357.295-.6.295l.213-3.054 5.56-5.022c.24-.213-.054-.334-.373-.121l-6.869 4.326-2.96-.924c-.64-.203-.658-.64.135-.954l11.566-4.458c.538-.196 1.006.128.832.941z"/>
      </svg>
    </a>
```
位置在这里（上下文）：
```vue
    <ProductQuickBuy
      v-if="quickBuyProduct"
      :product="quickBuyProduct"
      :visible="quickBuyVisible"
      @update:visible="quickBuyVisible = $event"
    />
  </div>
```
变成：

```vue
    <ProductQuickBuy
      v-if="quickBuyProduct"
      :product="quickBuyProduct"
      :visible="quickBuyVisible"
      @update:visible="quickBuyVisible = $event"
    />
    <!-- Floating Telegram Button -->
    <a
      v-if="appStore.config?.contact?.telegram"
      :href="appStore.config.contact.telegram"
      target="_blank"
      rel="noopener noreferrer"
      class="fixed right-5 bottom-24 md:bottom-8 z-50 w-12 h-12 rounded-full bg-[#2AABEE] text-white shadow-lg hover:shadow-xl hover:scale-110 transition-all flex items-center justify-center"
    >
      <svg class="w-6 h-6" fill="currentColor" viewBox="0 0 24 24">
        <path d="M12 0C5.373 0 0 5.373 0 12s5.373 12 12 12 12-5.373 12-12S18.627 0 12 0zm5.894 8.221l-1.97 9.28c-.145.658-.537.818-1.084.508l-3-2.21-1.446 1.394c-.14.18-.357.295-.6.295l.213-3.054 5.56-5.022c.24-.213-.054-.334-.373-.121l-6.869 4.326-2.96-.924c-.64-.203-.658-.64.135-.954l11.566-4.458c.538-.196 1.006.128.832.941z"/>
      </svg>
    </a>
  </div>
```

# Navbar.vue — 去掉快捷导航按钮
删除\src\components\Navbar.vue第 13-22 行的 Desktop Menu
```vue
      <!-- Desktop Menu -->
      <div class="hidden md:flex items-center space-x-1">
        <router-link v-for="item in menuItems" :key="item.path" :to="item.path"
          class="theme-nav-link text-sm relative group overflow-hidden flex items-center gap-1.5"
          active-class="theme-nav-link-active">
          <svg class="w-4 h-4 shrink-0 opacity-70" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.75" :d="item.icon" />
          </svg>
          <span class="relative z-10">{{ t(item.label) }}</span>
        </router-link>
      </div>
```

并删除 <script> 中不再使用的变量（第 222-234 行）
```vue
const isListMode = computed(() => appStore.config?.template_mode === 'list')
const allMenuItems = [
  { path: '/', label: 'nav.home', icon: '...' },
  { path: '/products', label: 'nav.products', icon: '...' },
  { path: '/blog', label: 'nav.blog', icon: '...' },
  { path: '/notice', label: 'nav.notice', icon: '...' },
  { path: '/about', label: 'nav.about', icon: '...' },
]
const menuItems = computed(() =>
  isListMode.value ? allMenuItems.filter(item => item.path !== '/products') : allMenuItems
)
```

# footer替换
```vue
<template>
  <footer
    class="relative theme-panel-strong theme-text-secondary border-t theme-border overflow-hidden">
    <div class="container mx-auto px-4 py-6 relative z-10">
      <div
        class="flex flex-col md:flex-row items-center justify-between gap-4 text-xs theme-text-muted">
        <p>&copy; {{ currentYear }} {{ brandSiteName }}. {{ t('footer.rights') }}</p>
        <div v-if="footerLinks.length" class="flex flex-wrap items-center gap-x-4 gap-y-1 justify-center md:justify-end">
          <a
            v-for="link in footerLinks"
            :key="link.name"
            :href="link.url || 'javascript:void(0)'"
            :target="link.url ? '_blank' : undefined"
            rel="noopener noreferrer"
            class="hover:text-gray-900 dark:hover:text-gray-400"
          >{{ link.name }}</a>
        </div>
      </div>
    </div>
  </footer>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useI18n } from 'vue-i18n'
import { useAppStore } from '../stores/app'

const { t } = useI18n()
const appStore = useAppStore()

const config = computed(() => appStore.config)

const brandSiteName = computed(() => {
  const siteName = config.value?.brand?.site_name
  return typeof siteName === 'string' && siteName.trim() ? siteName.trim() : '你的网站title'
})

const footerLinks = computed(() => {
  const links = config.value?.footer_links
  if (!Array.isArray(links)) return []
  return links.filter((item: any) => item && typeof item.name === 'string' && item.name.trim())
})

const currentYear = new Date().getFullYear()
</script>
```
