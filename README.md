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

3.删除 formatDate、goToPost、loadLatestPosts...  
找到这七个函数，全部删除：
```vue
const formatDate = (dateString: string) => { ... }
const goToPost = (slug: string) => { ... }
const loadLatestPosts = async () => { ... }
const navBuiltin = computed(() => (appStore.config?.nav_config as { builtin?: Record<string, boolean> } | undefined)?.builtin)
const blogEnabled = computed(() => navBuiltin.value?.blog !== false)
const noticeEnabled = computed(() => navBuiltin.value?.notice !== false)
const latestSectionVisible = computed(() => blogEnabled.value || noticeEnabled.value)
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
  </div>
```
公告区域插入到 Banner 后、精选商品前  

分隔线 + 最新动态整个 section 全部删除  

精选商品的 :class 改为 noticeContent ? 'pt-8' : showHeroSection ? 'pt-12' : 'pt-26'  

# HOME列表显示库存
user-main/src/composables/useProduct.ts的getStockStatusLabel 函数从：
```vue
  const getStockStatusLabel = (product: any) => {
    const status = product?.stock_status || ''
    if (status === 'unlimited') return t('products.stockStatus.unlimited')
    if (status === 'out_of_stock') return t('products.stockStatus.outOfStock')
    if (status === 'low_stock') {
      const count = Number(product?.fulfillment_type === 'manual' ? product?.manual_stock_available : product?.auto_stock_available)
      if (Number.isFinite(count) && count > 0) {
        return t('products.stockStatus.lowStockCount', { count })
      }
      return t('products.stockStatus.lowStock')
    }
    return t('products.stockStatus.inStock')
  }
```
改为：
```vue
const getStockStatusLabel = (product: any) => {
  const status = product?.stock_status || ''
  if (status === 'unlimited') return t('products.stockStatus.unlimited')

  const count = Number(
    product?.fulfillment_type === 'manual'
      ? product?.manual_stock_available
      : product?.auto_stock_available
  )

  if (Number.isFinite(count) && count >= 0) {
    return `库存剩余 ${count} 件`
  }

  return t('products.stockStatus.inStock')
}
```
# 移动端也显示库存
user-main\src\components\ProductListItem.vue 去掉移动端库存标签的 v-if 条件
```
        <span v-if="product.stock_status === 'out_of_stock' || product.stock_status === 'low_stock'"
          class="sm:hidden theme-badge text-[10px]"
          :class="getStockBadgeClass(product.stock_status)">
          {{ getStockStatusLabel(product) }}
        </span>
```
改成：
```
<span
  class="sm:hidden theme-badge text-[10px]"
  :class="getStockBadgeClass(product.stock_status)">
  {{ getStockStatusLabel(product) }}
</span>
```
删除
```
        <span v-if="product.category?.name" class="hidden sm:inline text-[11px] theme-text-muted uppercase tracking-wider truncate max-w-[80px] flex-shrink-0">
          {{ getLocalizedText(product.category.name) }}
        </span>
        <span v-if="product.category?.name" class="hidden sm:inline text-[11px] theme-text-muted opacity-30 flex-shrink-0">·</span>
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

8-10行加图标
```
      <router-link to="/" class="theme-wordmark group relative" :title="brandSiteName">
        <span class="theme-wordmark-text">{{ brandSiteName }}</span>
      </router-link>
```
改成
```
      <router-link to="/" class="theme-wordmark group relative flex items-center gap-2" :title="brandSiteName">
        <img src="/dj.svg" alt="Logo" class="h-5 w-5 shrink-0" />
        <span class="theme-wordmark-text">{{ brandSiteName }}</span>
      </router-link>
```

删除移动端导航
```
          <!-- Navigation items not in bottom nav -->
          <template v-for="item in mobileDrawerItems" :key="item.key">
            <router-link v-if="item.type === 'route'" :to="item.path" @click="showMobileMenu = false"
              class="block w-full text-left px-4 py-3 rounded-xl theme-nav-link text-sm min-h-[44px] flex items-center gap-3"
              active-class="theme-nav-link-active">
              <svg class="w-5 h-5 shrink-0 opacity-60" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.75" :d="item.icon" />
              </svg>
              {{ item.label.startsWith('nav.') ? t(item.label) : item.label }}
            </router-link>
            <a v-else :href="item.path" :target="item.target" rel="noopener noreferrer" @click="showMobileMenu = false"
              class="block w-full text-left px-4 py-3 rounded-xl theme-nav-link text-sm min-h-[44px] flex items-center gap-3">
              <svg class="w-5 h-5 shrink-0 opacity-60" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.75" :d="item.icon" />
              </svg>
              {{ item.label }}
            </a>
          </template>
```
搜索 dujiao-next 改成自己网站名称  

删除Script脚本部分
| 行号 | 内容 |
|---|---|
| 219 | `isListMode` |
| 222-226 | `builtinNavDefs` |
| 228-235 | `NavItem` 接口 |
| 237-243 | `navConfig` |
| 245-248 | `getCustomItemTitle` |
| 250-267 | `presetIcons` + `defaultIcon` |
| 269-287 | `buildCustomNavItems` |
| 289-297 | `buildBuiltinNavItems` |
| 299-309 | `menuItems` |
| 311-315 | `mobileDrawerItems` |

# footer替换
```vue
<template>
  <footer
    class="relative theme-panel-strong theme-text-secondary border-t theme-border overflow-hidden">
    <div class="container mx-auto px-4 py-6 relative z-10">
      <div
        class="flex flex-col md:flex-row items-center justify-between gap-4 text-xs theme-text-muted">
        <div class="space-y-1 text-center md:text-left">
          <p>&copy; {{ currentYear }} {{ brandSiteName }}. {{ t('footer.rights') }}</p>
          <p class="flex items-center justify-center gap-1 md:justify-start">
            <span>Open Source: Dujiao-Next ·</span>
            <a
              href="https://github.com/dujiao-next"
              target="_blank"
              rel="noopener noreferrer"
              class="inline-flex items-center gap-1 hover:text-gray-900 dark:hover:text-gray-400"
            >
              <svg class="h-3.5 w-3.5" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
                <path d="M12 .5C5.648.5.5 5.648.5 12c0 5.084 3.292 9.4 7.86 10.922.575.106.784-.25.784-.556 0-.273-.01-1-.016-1.962-3.197.694-3.872-1.54-3.872-1.54-.522-1.326-1.274-1.678-1.274-1.678-1.042-.713.079-.699.079-.699 1.152.081 1.758 1.183 1.758 1.183 1.024 1.755 2.688 1.248 3.343.954.104-.742.401-1.248.73-1.535-2.552-.29-5.236-1.276-5.236-5.678 0-1.254.448-2.28 1.182-3.084-.118-.29-.512-1.457.112-3.04 0 0 .964-.308 3.158 1.178a10.98 10.98 0 0 1 2.876-.387c.976.004 1.96.132 2.878.387 2.192-1.486 3.154-1.178 3.154-1.178.626 1.583.232 2.75.114 3.04.736.804 1.18 1.83 1.18 3.084 0 4.413-2.688 5.384-5.248 5.668.412.354.78 1.052.78 2.12 0 1.53-.014 2.764-.014 3.14 0 .31.206.668.79.554C20.212 21.396 23.5 17.083 23.5 12 23.5 5.648 18.352.5 12 .5Z" />
              </svg>
              <span>https://github.com/dujiao-next</span>
            </a>
          </p>
        </div>
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
  return typeof siteName === 'string' && siteName.trim() ? siteName.trim() : 'Dujiao-Next'
})

const footerLinks = computed(() => {
  const links = config.value?.footer_links
  if (!Array.isArray(links)) return []
  return links.filter((item: any) => item && typeof item.name === 'string' && item.name.trim())
})

const currentYear = new Date().getFullYear()
</script>
```
# api文档修改隐藏前端销量
\dujiao-next-main\internal\http\handlers\public\public.go中PublicSKUView
添加
```
	AutoStockSold        *models.Money `json:"auto_stock_sold,omitempty"`
	ManualStockSold      *models.Money `json:"manual_stock_sold,omitempty"`
```
PublicProductView中添加
```
	AutoStockSold        *struct{}              `json:"auto_stock_sold,omitempty"`
	ManualStockSold      *struct{}              `json:"manual_stock_sold,omitempty"`
```

删除item.AutoStockSold = 0和item.AutoStockSold = autoSold  
api打包命令 $env:CGO_ENABLED="0"; $env:GOOS="linux"; $env:GOARCH="amd64"; go build -trimpath -tags release -ldflags="-s -w" -o dujiao-next ./cmd/server
