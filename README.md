# 独角next前台魔改

# HOME页
1.在 <script setup> 的 import 区域，import { getImageUrl } 那行后面加一行：
```vue
import { processHtmlForDisplay } from '../utils/content'
```

2.替换变量声明  
找到：
```vue
const products = ref<any[]>([])
const posts = ref<any[]>([])
const quickBuyProduct = ref<any>(null)
const quickBuyVisible = ref(false)
```
替换为：
```vue
const products = ref<any[]>([])
const quickBuyProduct = ref<any>(null)
const quickBuyVisible = ref(false)
const noticeContent = ref('')

const loadLatestNotice = async () => {
  try {
    const response = await postAPI.list({ type: 'notice', page: 1, page_size: 1 })
    const notices = response.data.data || []
    if (notices.length > 0) {
      const notice = notices[0]
      const detailRes = await postAPI.detail(notice.slug)
      const post = detailRes.data.data
      const content = getLocalizedText(post.content)
      if (content && content.trim()) {
        noticeContent.value = processHtmlForDisplay(content)
      }
    }
  } catch (error) {
    console.error('Failed to load latest notice:', error)
  }
}
```

3.删除 formatDate、goToPost、loadLatestPosts  
找到这三个函数，全部删除：
```vue
const formatDate = (dateString: string) => { ... }
const goToPost = (slug: string) => { ... }
const loadLatestPosts = async () => { ... }
```

4.修改 onMounted 生命周期  
找到：
```vue
onMounted(async () => {
  if (templateMode.value === 'list') {
    await Promise.all([loadBanners(), listInitialize()])
  } else {
    await Promise.all([loadBanners(), loadFeaturedProducts(), loadLatestPosts()])
  }
})
```
替换为：
```vue
onMounted(async () => {
  if (templateMode.value === 'list') {
    await Promise.all([loadBanners(), listInitialize(), loadLatestNotice()])
  } else {
    await Promise.all([loadBanners(), loadFeaturedProducts(), loadLatestNotice()])
  }
})
```

5.LIST 模式模板 — 在 Banner 和商品列表之间插入公告
找到 list 模式中 Banner </section> 后面的：
```vue
      <!-- Main: Left Categories + Right Product List -->
      <section class="relative z-10 pb-6" :class="showHeroSection ? 'pt-6' : 'pt-24'">
```
替换为：
```vue
      <!-- Notice Content (List Mode) -->
      <section v-if="noticeContent" class="relative z-10" :class="showHeroSection ? 'pt-12' : 'pt-26'">
        <div class="container mx-auto px-4">
          <div class="rounded-xl border theme-panel p-5">
            <div class="flex items-center gap-2 mb-3">
              <span class="w-1 h-5 rounded-full theme-accent-stick flex-shrink-0"></span>
              <h3 class="text-base font-semibold theme-text-primary">{{ t('nav.notice') }}</h3>
            </div>
            <div
              v-html="noticeContent"
              class="text-sm theme-text-secondary prose prose-sm max-w-none dark:prose-invert prose-p:my-1 prose-p:leading-relaxed"
            ></div>
          </div>
        </div>
      </section>

      <!-- Main: Left Categories + Right Product List -->
      <section class="relative z-10 pb-6" :class="showHeroSection || noticeContent ? 'pt-6' : 'pt-24'">
```

6.CARD 模式模板 — 删除"最新动态"，加入公告 + Telegram
找到 card 模式中精选商品 </section> 之后的所有内容，直到 </template> 结束：
```vue
    <!-- 原来的 282 行 -->
    <section id="featured" class="relative z-10 pb-14" :class="showHeroSection ? 'pt-14' : 'pt-32 md:pt-36'">
      ...精选商品...
    </section>

    <hr class="theme-section-divider ..." />

    <section class="relative z-10 py-12">
      ...最新动态（博客/公告链接 + 文章列表）...
    </section>
    </template>

    <ProductQuickBuy ... />
  </div>
```
替换为：
```vue
    <!-- Notice Content -->
    <section v-if="noticeContent" class="relative z-10" :class="showHeroSection ? 'pt-12' : 'pt-26'">
      <div class="container mx-auto px-4">
        <div class="rounded-xl border theme-panel p-5">
          <div class="flex items-center gap-2 mb-3">
            <span class="w-1 h-5 rounded-full theme-accent-stick flex-shrink-0"></span>
            <h3 class="text-base font-semibold theme-text-primary">{{ t('nav.notice') }}</h3>
          </div>
          <div
            v-html="noticeContent"
            class="text-sm theme-text-secondary prose prose-sm max-w-none dark:prose-invert prose-p:my-1 prose-p:leading-relaxed"
          ></div>
        </div>
      </div>
    </section>

    <section id="featured" class="relative z-10 pb-14" :class="noticeContent ? 'pt-8' : showHeroSection ? 'pt-12' : 'pt-26'">
      ...精选商品（保持不变）...
    </section>

    </template>

    <ProductQuickBuy
      v-if="quickBuyProduct"
      :product="quickBuyProduct"
      :visible="quickBuyVisible"
      @update:visible="quickBuyVisible = $event"
    />

    <a
      v-if="appStore.config?.contact?.telegram"
      :href="appStore.config.contact.telegram"
      target="_blank"
      rel="noopener noreferrer"
      class="fixed right-5 bottom-24 md:bottom-8 z-50 w-16 h-16 rounded-full bg-[#2AABEE] text-white shadow-lg hover:shadow-xl hover:scale-110 transition-all flex items-center justify-center"
      aria-label="Telegram"
    >
      <svg class="w-9 h-9" fill="currentColor" viewBox="0 0 24 24">
        <path d="M12 0C5.373 0 0 5.373 0 12s5.373 12 12 12 12-5.373 12-12S18.627 0 12 0zm5.894 8.221l-1.97 9.28c-.145.658-.537.818-1.084.508l-3-2.21-1.446 1.394c-.14.18-.357.295-.6.295l.213-3.054 5.56-5.022c.24-.213-.054-.334-.373-.121l-6.869 4.326-2.96-.924c-.64-.203-.658-.64.135-.954l11.566-4.458c.538-.196 1.006.128.832.941z"/>
      </svg>
    </a>
  </div>
```
公告区域插入到 Banner 后、精选商品前  

分隔线 + 最新动态整个 section 全部删除  

精选商品的 :class 改为 noticeContent ? 'pt-8' : showHeroSection ? 'pt-12' : 'pt-26'  


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

8-10行
```
      <router-link to="/" class="theme-wordmark group relative" :title="brandSiteName">
        <span class="theme-wordmark-text">{{ brandSiteName }}</span>
      </router-link>
```
改成
```
      <router-link to="/" class="flex items-center gap-2 group" :title="brandSiteName">
        <img src="/dj.svg" alt="Logo" class="h-7 w-7 shrink-0" />
        <span class="theme-wordmark">
          <span class="theme-wordmark-text">{{ brandSiteName }}</span>
        </span>
      </router-link>
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
