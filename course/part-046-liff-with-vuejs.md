# Part 46: LIFF กับ Vue.js - สร้าง Mini App ด้วย Vue 3

## บทนำ

LIFF (LINE Front-end Framework) เป็น platform ที่ช่วยให้เราสร้าง web application ที่รันใน LINE app ได้ การใช้ Vue 3 ร่วมกับ LIFF ทำให้เราสามารถสร้าง Single Page Application (SPA) ที่มีประสิทธิภาพสูงและ user experience ที่ดีมากขึ้น

ในบทนี้เราจะเรียนรู้:
- การตั้งค่า Vue 3 + Vite + LIFF
- การสร้าง LIFF Plugin สำหรับ Vue
- การใช้ Composition API กับ LIFF
- การสร้าง Composables สำหรับ LIFF features ต่างๆ
- Vue Router กับ LIFF navigation
- Pinia store สำหรับ LIFF state
- สร้าง Mini Vue LIFF App ครบทั้งระบบ

---

## 1. การตั้งค่า Project

### 1.1 สร้าง Vite + Vue 3 Project

```bash
# สร้าง project ใหม่
npm create vite@latest my-liff-app -- --template vue

# เข้าไปใน directory
cd my-liff-app

# ติดตั้ง dependencies พื้นฐาน
npm install

# ติดตั้ง LIFF SDK
npm install @line/liff

# ติดตั้ง Vue Router
npm install vue-router@4

# ติดตั้ง Pinia (state management)
npm install pinia

# ติดตั้ง Axios สำหรับ API calls
npm install axios

# ติดตั้ง VueUse (utility composables)
npm install @vueuse/core

# ติดตั้ง dev dependencies
npm install -D @types/node sass autoprefixer postcss
```

### 1.2 โครงสร้าง Project

```
my-liff-app/
├── public/
│   └── favicon.ico
├── src/
│   ├── assets/
│   │   └── styles/
│   │       ├── main.scss
│   │       └── variables.scss
│   ├── components/
│   │   ├── common/
│   │   │   ├── LoadingSpinner.vue
│   │   │   ├── ErrorMessage.vue
│   │   │   └── UserAvatar.vue
│   │   ├── profile/
│   │   │   ├── ProfileCard.vue
│   │   │   └── ProfileEditor.vue
│   │   └── layout/
│   │       ├── AppHeader.vue
│   │       └── AppFooter.vue
│   ├── composables/
│   │   ├── useLiff.ts
│   │   ├── useLiffProfile.ts
│   │   ├── useLiffShare.ts
│   │   ├── useLiffScanner.ts
│   │   └── useLiffPermission.ts
│   ├── plugins/
│   │   └── liff.ts
│   ├── router/
│   │   └── index.ts
│   ├── stores/
│   │   ├── liff.ts
│   │   └── user.ts
│   ├── views/
│   │   ├── HomeView.vue
│   │   ├── ProfileView.vue
│   │   ├── ShareView.vue
│   │   └── NotFoundView.vue
│   ├── App.vue
│   └── main.ts
├── index.html
├── vite.config.ts
├── tsconfig.json
└── package.json
```

### 1.3 ไฟล์ vite.config.ts

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src'),
    },
  },
  server: {
    port: 3000,
    https: true, // LIFF ต้องการ HTTPS
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['vue', 'vue-router', 'pinia'],
          liff: ['@line/liff'],
        },
      },
    },
  },
  define: {
    __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
  },
})
```

### 1.4 ตั้งค่า Environment Variables

```bash
# .env.development
VITE_LIFF_ID=your-liff-id-here
VITE_API_BASE_URL=https://api.yourserver.com
VITE_APP_ENV=development

# .env.production
VITE_LIFF_ID=your-production-liff-id
VITE_API_BASE_URL=https://api.yourserver.com
VITE_APP_ENV=production
```

---

## 2. LIFF Plugin สำหรับ Vue

### 2.1 สร้าง LIFF Plugin

```typescript
// src/plugins/liff.ts
import { App, Plugin } from 'vue'
import liff from '@line/liff'

export interface LiffPluginOptions {
  liffId: string
  mock?: boolean
  stubEnabled?: boolean
}

export interface LiffInstance {
  init: (config: { liffId: string }) => Promise<void>
  isLoggedIn: () => boolean
  login: (config?: { redirectUri?: string }) => void
  logout: () => void
  getProfile: () => Promise<LiffProfile>
  getAccessToken: () => string | null
  getIDToken: () => string | null
  getContext: () => LiffContext | null
  sendMessages: (messages: object[]) => Promise<void>
  shareTargetPicker: (messages: object[]) => Promise<void>
  scanCodeV2: () => Promise<{ value: string }>
  openWindow: (params: { url: string; external?: boolean }) => void
  closeWindow: () => void
  getOS: () => string
  getVersion: () => string
  isInClient: () => boolean
  isApiAvailable: (api: string) => boolean
  ready: Promise<void>
}

export interface LiffProfile {
  userId: string
  displayName: string
  pictureUrl?: string
  statusMessage?: string
}

export interface LiffContext {
  type: string
  utouId?: string
  roomId?: string
  groupId?: string
  userId?: string
  accessTokenHash?: string
  availability?: {
    shareTargetPicker: {
      permission: boolean
      minVer: string
    }
    multipleLiffTransition: {
      permission: boolean
      minVer: string
    }
    subwindow: {
      permission: boolean
      minVer: string
    }
  }
}

// Symbol สำหรับ inject
export const LIFF_KEY = Symbol('liff')

const LiffPlugin: Plugin = {
  install(app: App, options: LiffPluginOptions) {
    const { liffId } = options

    // เพิ่ม liff ลงใน global properties
    app.config.globalProperties.$liff = liff

    // Provide liff สำหรับ inject
    app.provide(LIFF_KEY, liff)

    // เริ่มต้น LIFF เมื่อ app mount
    const initLiff = async () => {
      try {
        await liff.init({ liffId })
        console.log('LIFF initialized successfully')
      } catch (error) {
        console.error('LIFF initialization failed:', error)
      }
    }

    initLiff()
  },
}

export default LiffPlugin
```

### 2.2 ลงทะเบียน Plugin ใน main.ts

```typescript
// src/main.ts
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './router'
import LiffPlugin from './plugins/liff'

const app = createApp(App)

// ติดตั้ง Pinia
const pinia = createPinia()
app.use(pinia)

// ติดตั้ง Router
app.use(router)

// ติดตั้ง LIFF Plugin
app.use(LiffPlugin, {
  liffId: import.meta.env.VITE_LIFF_ID,
})

app.mount('#app')
```

---

## 3. Composables สำหรับ LIFF

### 3.1 useLiff - Core Composable

```typescript
// src/composables/useLiff.ts
import { ref, computed, onMounted } from 'vue'
import liff from '@line/liff'

export interface LiffState {
  isInitialized: boolean
  isLoggedIn: boolean
  isInClient: boolean
  os: string
  version: string
  language: string
  accessToken: string | null
  context: any | null
}

export function useLiff() {
  const state = ref<LiffState>({
    isInitialized: false,
    isLoggedIn: false,
    isInClient: false,
    os: '',
    version: '',
    language: '',
    accessToken: null,
    context: null,
  })

  const isLoading = ref(false)
  const error = ref<string | null>(null)

  // Computed properties
  const isReady = computed(() => state.value.isInitialized)
  const canShare = computed(() => {
    if (!state.value.isInitialized) return false
    return liff.isApiAvailable('shareTargetPicker')
  })
  const canScan = computed(() => {
    if (!state.value.isInitialized) return false
    return liff.isApiAvailable('scanCodeV2')
  })

  // Initialize LIFF
  const initialize = async (liffId?: string) => {
    isLoading.value = true
    error.value = null

    try {
      const id = liffId || import.meta.env.VITE_LIFF_ID

      if (!id) {
        throw new Error('LIFF ID is required')
      }

      await liff.init({ liffId: id })

      state.value = {
        isInitialized: true,
        isLoggedIn: liff.isLoggedIn(),
        isInClient: liff.isInClient(),
        os: liff.getOS(),
        version: liff.getVersion(),
        language: liff.getLanguage(),
        accessToken: liff.getAccessToken(),
        context: liff.getContext(),
      }

      return true
    } catch (err: any) {
      error.value = err.message || 'LIFF initialization failed'
      console.error('LIFF init error:', err)
      return false
    } finally {
      isLoading.value = false
    }
  }

  // Login
  const login = (redirectUri?: string) => {
    if (!state.value.isInitialized) {
      console.warn('LIFF is not initialized')
      return
    }

    liff.login({ redirectUri })
  }

  // Logout
  const logout = () => {
    if (!state.value.isInitialized) return

    liff.logout()
    state.value.isLoggedIn = false
    state.value.accessToken = null
  }

  // Close window
  const closeWindow = () => {
    if (liff.isInClient()) {
      liff.closeWindow()
    } else {
      window.close()
    }
  }

  // Open external URL
  const openWindow = (url: string, external = true) => {
    liff.openWindow({ url, external })
  }

  // Send messages to chat
  const sendMessages = async (messages: object[]) => {
    if (!state.value.isLoggedIn) {
      throw new Error('User must be logged in to send messages')
    }

    try {
      await liff.sendMessages(messages)
      return true
    } catch (err: any) {
      error.value = err.message
      throw err
    }
  }

  onMounted(async () => {
    if (!state.value.isInitialized) {
      await initialize()
    }
  })

  return {
    state,
    isLoading,
    error,
    isReady,
    canShare,
    canScan,
    initialize,
    login,
    logout,
    closeWindow,
    openWindow,
    sendMessages,
  }
}
```

### 3.2 useLiffProfile - Profile Composable

```typescript
// src/composables/useLiffProfile.ts
import { ref, computed } from 'vue'
import liff from '@line/liff'

export interface LineProfile {
  userId: string
  displayName: string
  pictureUrl?: string
  statusMessage?: string
  email?: string
}

export function useLiffProfile() {
  const profile = ref<LineProfile | null>(null)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  const displayName = computed(() => profile.value?.displayName || 'ไม่ระบุชื่อ')
  const avatarUrl = computed(
    () => profile.value?.pictureUrl || 'https://via.placeholder.com/150'
  )
  const userId = computed(() => profile.value?.userId || null)

  const fetchProfile = async () => {
    if (!liff.isLoggedIn()) {
      error.value = 'User is not logged in'
      return null
    }

    isLoading.value = true
    error.value = null

    try {
      const data = await liff.getProfile()
      profile.value = {
        userId: data.userId,
        displayName: data.displayName,
        pictureUrl: data.pictureUrl,
        statusMessage: data.statusMessage,
      }

      // ดึง email ถ้ามี permission
      try {
        const idToken = liff.getIDToken()
        if (idToken) {
          const decodedToken = parseJwt(idToken)
          profile.value.email = decodedToken.email
        }
      } catch {
        // email อาจไม่ได้ขอ permission
      }

      return profile.value
    } catch (err: any) {
      error.value = err.message || 'Failed to fetch profile'
      return null
    } finally {
      isLoading.value = false
    }
  }

  const clearProfile = () => {
    profile.value = null
  }

  return {
    profile,
    isLoading,
    error,
    displayName,
    avatarUrl,
    userId,
    fetchProfile,
    clearProfile,
  }
}

// Helper: decode JWT
function parseJwt(token: string) {
  try {
    const base64Url = token.split('.')[1]
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/')
    const jsonPayload = decodeURIComponent(
      atob(base64)
        .split('')
        .map((c) => '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2))
        .join('')
    )
    return JSON.parse(jsonPayload)
  } catch {
    return {}
  }
}
```

### 3.3 useLiffShare - Share Composable

```typescript
// src/composables/useLiffShare.ts
import { ref } from 'vue'
import liff from '@line/liff'

export interface FlexMessage {
  type: 'flex'
  altText: string
  contents: object
}

export interface TextMessage {
  type: 'text'
  text: string
}

export type ShareMessage = FlexMessage | TextMessage

export function useLiffShare() {
  const isSharing = ref(false)
  const shareError = ref<string | null>(null)

  // Share via Target Picker
  const shareToFriend = async (messages: ShareMessage[]) => {
    if (!liff.isApiAvailable('shareTargetPicker')) {
      shareError.value = 'shareTargetPicker API is not available'
      return false
    }

    isSharing.value = true
    shareError.value = null

    try {
      const result = await liff.shareTargetPicker(messages)

      if (result) {
        // User เลือก target แล้ว
        console.log('Share successful, requestId:', result.status)
        return true
      } else {
        // User ยกเลิก
        console.log('Share cancelled')
        return false
      }
    } catch (err: any) {
      shareError.value = err.message || 'Share failed'
      return false
    } finally {
      isSharing.value = false
    }
  }

  // Send message กลับไปยัง chat ที่เปิด LIFF
  const sendToChat = async (messages: ShareMessage[]) => {
    if (!liff.isLoggedIn()) {
      shareError.value = 'User must be logged in'
      return false
    }

    try {
      await liff.sendMessages(messages)
      return true
    } catch (err: any) {
      shareError.value = err.message
      return false
    }
  }

  // สร้าง Flex Message สำหรับแชร์ Product
  const createProductMessage = (product: {
    name: string
    price: number
    imageUrl: string
    url: string
  }): FlexMessage => {
    return {
      type: 'flex',
      altText: `สินค้า: ${product.name}`,
      contents: {
        type: 'bubble',
        hero: {
          type: 'image',
          url: product.imageUrl,
          size: 'full',
          aspectRatio: '20:13',
          aspectMode: 'cover',
          action: {
            type: 'uri',
            uri: product.url,
          },
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: product.name,
              weight: 'bold',
              size: 'xl',
            },
            {
              type: 'box',
              layout: 'vertical',
              margin: 'lg',
              spacing: 'sm',
              contents: [
                {
                  type: 'box',
                  layout: 'baseline',
                  spacing: 'sm',
                  contents: [
                    {
                      type: 'text',
                      text: 'ราคา',
                      color: '#aaaaaa',
                      size: 'sm',
                      flex: 1,
                    },
                    {
                      type: 'text',
                      text: `฿${product.price.toLocaleString()}`,
                      wrap: true,
                      color: '#666666',
                      size: 'sm',
                      flex: 5,
                    },
                  ],
                },
              ],
            },
          ],
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          spacing: 'sm',
          contents: [
            {
              type: 'button',
              style: 'link',
              height: 'sm',
              action: {
                type: 'uri',
                label: 'ดูสินค้า',
                uri: product.url,
              },
            },
          ],
          flex: 0,
        },
      },
    }
  }

  return {
    isSharing,
    shareError,
    shareToFriend,
    sendToChat,
    createProductMessage,
  }
}
```

### 3.4 useLiffScanner - QR Scanner Composable

```typescript
// src/composables/useLiffScanner.ts
import { ref } from 'vue'
import liff from '@line/liff'

export function useLiffScanner() {
  const scanResult = ref<string | null>(null)
  const isScanning = ref(false)
  const scanError = ref<string | null>(null)

  const scanQRCode = async () => {
    if (!liff.isApiAvailable('scanCodeV2')) {
      scanError.value = 'QR Scanner is not available on this device'
      return null
    }

    isScanning.value = true
    scanError.value = null

    try {
      const result = await liff.scanCodeV2()
      scanResult.value = result.value
      return result.value
    } catch (err: any) {
      if (err.code === 'CANCEL') {
        // User ยกเลิกการ scan
        return null
      }
      scanError.value = err.message || 'Scan failed'
      return null
    } finally {
      isScanning.value = false
    }
  }

  const clearScanResult = () => {
    scanResult.value = null
  }

  return {
    scanResult,
    isScanning,
    scanError,
    scanQRCode,
    clearScanResult,
  }
}
```

### 3.5 useLiffPermission - Permission Composable

```typescript
// src/composables/useLiffPermission.ts
import { ref, computed } from 'vue'
import liff from '@line/liff'

export type LiffPermission = 'profile' | 'openid' | 'email'

export function useLiffPermission() {
  const permissions = ref<string[]>([])
  const isLoading = ref(false)

  const hasProfile = computed(() => permissions.value.includes('profile'))
  const hasEmail = computed(() => permissions.value.includes('email'))
  const hasOpenId = computed(() => permissions.value.includes('openid'))

  // ตรวจสอบ permission ที่มี
  const checkPermissions = () => {
    try {
      const context = liff.getContext()
      if (context) {
        // ตรวจสอบจาก token หรือ context
        const token = liff.getAccessToken()
        if (token) {
          // decode token เพื่อดู scope
          const idToken = liff.getIDToken()
          if (idToken) {
            const decoded = parseJwt(idToken)
            // permissions จะอยู่ใน amr หรือ scope
          }
        }
      }
    } catch (err) {
      console.error('Permission check failed:', err)
    }
  }

  return {
    permissions,
    isLoading,
    hasProfile,
    hasEmail,
    hasOpenId,
    checkPermissions,
  }
}

function parseJwt(token: string) {
  try {
    const base64Url = token.split('.')[1]
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/')
    return JSON.parse(atob(base64))
  } catch {
    return {}
  }
}
```

---

## 4. Pinia Store สำหรับ LIFF State

### 4.1 LIFF Store

```typescript
// src/stores/liff.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import liff from '@line/liff'

export const useLiffStore = defineStore('liff', () => {
  // State
  const isInitialized = ref(false)
  const isLoggedIn = ref(false)
  const isInClient = ref(false)
  const os = ref('')
  const version = ref('')
  const language = ref('')
  const accessToken = ref<string | null>(null)
  const idToken = ref<string | null>(null)
  const context = ref<any | null>(null)
  const initError = ref<string | null>(null)

  // Getters (Computed)
  const isReady = computed(() => isInitialized.value)
  const isAuthenticated = computed(() => isInitialized.value && isLoggedIn.value)
  const isMobile = computed(() => ['ios', 'android'].includes(os.value.toLowerCase()))
  const canUseShareTargetPicker = computed(() => {
    if (!isInitialized.value) return false
    return liff.isApiAvailable('shareTargetPicker')
  })
  const canUseScanCode = computed(() => {
    if (!isInitialized.value) return false
    return liff.isApiAvailable('scanCodeV2')
  })

  // Actions
  const init = async (liffId: string) => {
    try {
      await liff.init({ liffId })

      isInitialized.value = true
      isLoggedIn.value = liff.isLoggedIn()
      isInClient.value = liff.isInClient()
      os.value = liff.getOS()
      version.value = liff.getVersion()
      language.value = liff.getLanguage()
      accessToken.value = liff.getAccessToken()
      idToken.value = liff.getIDToken()
      context.value = liff.getContext()
      initError.value = null

      return true
    } catch (error: any) {
      initError.value = error.message
      console.error('LIFF init failed:', error)
      return false
    }
  }

  const login = (redirectUri?: string) => {
    liff.login({ redirectUri })
  }

  const logout = () => {
    liff.logout()
    isLoggedIn.value = false
    accessToken.value = null
    idToken.value = null
  }

  const refreshToken = () => {
    accessToken.value = liff.getAccessToken()
    idToken.value = liff.getIDToken()
  }

  return {
    // State
    isInitialized,
    isLoggedIn,
    isInClient,
    os,
    version,
    language,
    accessToken,
    idToken,
    context,
    initError,
    // Getters
    isReady,
    isAuthenticated,
    isMobile,
    canUseShareTargetPicker,
    canUseScanCode,
    // Actions
    init,
    login,
    logout,
    refreshToken,
  }
})
```

### 4.2 User Store

```typescript
// src/stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import liff from '@line/liff'
import axios from 'axios'

export interface UserProfile {
  userId: string
  displayName: string
  pictureUrl?: string
  statusMessage?: string
  email?: string
}

export interface UserPoints {
  total: number
  rank: string
  history: PointHistory[]
}

export interface PointHistory {
  id: string
  amount: number
  type: 'earn' | 'redeem'
  description: string
  createdAt: string
}

export const useUserStore = defineStore('user', () => {
  const profile = ref<UserProfile | null>(null)
  const points = ref<UserPoints | null>(null)
  const isLoadingProfile = ref(false)
  const isLoadingPoints = ref(false)
  const profileError = ref<string | null>(null)

  const displayName = computed(() => profile.value?.displayName || 'ผู้ใช้')
  const avatarUrl = computed(
    () =>
      profile.value?.pictureUrl ||
      `https://api.dicebear.com/7.x/avataaars/svg?seed=${profile.value?.userId}`
  )
  const totalPoints = computed(() => points.value?.total || 0)

  const fetchProfile = async () => {
    if (!liff.isLoggedIn()) {
      profileError.value = 'Please login first'
      return
    }

    isLoadingProfile.value = true
    profileError.value = null

    try {
      const lineProfile = await liff.getProfile()
      profile.value = {
        userId: lineProfile.userId,
        displayName: lineProfile.displayName,
        pictureUrl: lineProfile.pictureUrl,
        statusMessage: lineProfile.statusMessage,
      }

      // ดึง email จาก ID token
      const idToken = liff.getIDToken()
      if (idToken) {
        const decoded = decodeJWT(idToken)
        profile.value.email = decoded.email
      }
    } catch (error: any) {
      profileError.value = error.message
    } finally {
      isLoadingProfile.value = false
    }
  }

  const fetchPoints = async () => {
    if (!profile.value?.userId) return

    isLoadingPoints.value = true
    try {
      const accessToken = liff.getAccessToken()
      const response = await axios.get(
        `${import.meta.env.VITE_API_BASE_URL}/users/${profile.value.userId}/points`,
        {
          headers: { Authorization: `Bearer ${accessToken}` },
        }
      )
      points.value = response.data
    } catch (error) {
      console.error('Failed to fetch points:', error)
    } finally {
      isLoadingPoints.value = false
    }
  }

  const clearUser = () => {
    profile.value = null
    points.value = null
  }

  return {
    profile,
    points,
    isLoadingProfile,
    isLoadingPoints,
    profileError,
    displayName,
    avatarUrl,
    totalPoints,
    fetchProfile,
    fetchPoints,
    clearUser,
  }
})

function decodeJWT(token: string) {
  try {
    const base64Url = token.split('.')[1]
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/')
    return JSON.parse(atob(base64))
  } catch {
    return {}
  }
}
```

---

## 5. Vue Router กับ LIFF

### 5.1 Router Configuration

```typescript
// src/router/index.ts
import { createRouter, createWebHistory, RouteRecordRaw } from 'vue-router'
import { useLiffStore } from '@/stores/liff'
import { useUserStore } from '@/stores/user'

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    name: 'home',
    component: () => import('@/views/HomeView.vue'),
    meta: {
      title: 'หน้าหลัก',
      requiresAuth: false,
    },
  },
  {
    path: '/profile',
    name: 'profile',
    component: () => import('@/views/ProfileView.vue'),
    meta: {
      title: 'โปรไฟล์',
      requiresAuth: true,
    },
  },
  {
    path: '/share',
    name: 'share',
    component: () => import('@/views/ShareView.vue'),
    meta: {
      title: 'แชร์',
      requiresAuth: true,
    },
  },
  {
    path: '/points',
    name: 'points',
    component: () => import('@/views/PointsView.vue'),
    meta: {
      title: 'คะแนนสะสม',
      requiresAuth: true,
    },
  },
  {
    path: '/scan',
    name: 'scan',
    component: () => import('@/views/ScanView.vue'),
    meta: {
      title: 'สแกน QR',
      requiresAuth: true,
    },
  },
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: () => import('@/views/NotFoundView.vue'),
  },
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition
    }
    return { top: 0 }
  },
})

// Navigation Guard
router.beforeEach(async (to, from, next) => {
  const liffStore = useLiffStore()
  const userStore = useUserStore()

  // อัปเดต page title
  if (to.meta.title) {
    document.title = `${to.meta.title} | LINE Mini App`
  }

  // ตรวจสอบ authentication
  if (to.meta.requiresAuth) {
    if (!liffStore.isAuthenticated) {
      // ถ้าไม่ได้ login ให้ redirect ไป login
      liffStore.login(window.location.href)
      return
    }

    // ถ้ายังไม่มี profile ให้ fetch
    if (!userStore.profile) {
      await userStore.fetchProfile()
    }
  }

  next()
})

export default router
```

---

## 6. Components

### 6.1 App.vue - Root Component

```vue
<!-- src/App.vue -->
<template>
  <div id="app" :class="{ 'in-client': liffStore.isInClient }">
    <!-- Loading Screen -->
    <Transition name="fade">
      <div v-if="isInitializing" class="loading-screen">
        <div class="loading-content">
          <img src="@/assets/logo.png" alt="Logo" class="loading-logo" />
          <div class="loading-spinner" />
          <p class="loading-text">กำลังโหลด...</p>
        </div>
      </div>
    </Transition>

    <!-- Error Screen -->
    <div v-if="initError" class="error-screen">
      <div class="error-content">
        <span class="error-icon">⚠️</span>
        <h2>เกิดข้อผิดพลาด</h2>
        <p>{{ initError }}</p>
        <button @click="retryInit" class="retry-btn">ลองอีกครั้ง</button>
      </div>
    </div>

    <!-- Main App -->
    <template v-if="!isInitializing && !initError">
      <AppHeader />
      <main class="main-content">
        <RouterView v-slot="{ Component }">
          <Transition name="slide-fade" mode="out-in">
            <component :is="Component" />
          </Transition>
        </RouterView>
      </main>
      <AppFooter />
    </template>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { RouterView } from 'vue-router'
import { useLiffStore } from '@/stores/liff'
import AppHeader from '@/components/layout/AppHeader.vue'
import AppFooter from '@/components/layout/AppFooter.vue'

const liffStore = useLiffStore()
const isInitializing = ref(true)
const initError = ref<string | null>(null)

const initApp = async () => {
  isInitializing.value = true
  initError.value = null

  const success = await liffStore.init(import.meta.env.VITE_LIFF_ID)

  if (!success) {
    initError.value = liffStore.initError || 'ไม่สามารถเริ่มต้น LIFF ได้'
  }

  isInitializing.value = false
}

const retryInit = async () => {
  await initApp()
}

onMounted(async () => {
  await initApp()
})
</script>

<style lang="scss">
#app {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: var(--bg-color);
  color: var(--text-color);

  &.in-client {
    // Styles เฉพาะเมื่ออยู่ใน LINE app
    padding-bottom: env(safe-area-inset-bottom);
  }
}

.loading-screen {
  position: fixed;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-color);
  z-index: 9999;

  .loading-content {
    text-align: center;

    .loading-logo {
      width: 80px;
      height: 80px;
      margin-bottom: 24px;
    }

    .loading-spinner {
      width: 40px;
      height: 40px;
      border: 3px solid var(--border-color);
      border-top-color: var(--primary-color);
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
      margin: 0 auto 16px;
    }

    .loading-text {
      color: var(--text-secondary);
      font-size: 14px;
    }
  }
}

.error-screen {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  padding: 24px;

  .error-content {
    text-align: center;
    max-width: 300px;

    .error-icon {
      font-size: 48px;
      display: block;
      margin-bottom: 16px;
    }

    h2 {
      margin-bottom: 8px;
      color: var(--error-color);
    }

    p {
      color: var(--text-secondary);
      margin-bottom: 24px;
    }

    .retry-btn {
      padding: 12px 32px;
      background: var(--primary-color);
      color: white;
      border: none;
      border-radius: 24px;
      font-size: 16px;
      cursor: pointer;

      &:active {
        opacity: 0.8;
      }
    }
  }
}

.main-content {
  flex: 1;
  overflow-y: auto;
}

// Transitions
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-fade-enter-active {
  transition: all 0.3s ease-out;
}

.slide-fade-leave-active {
  transition: all 0.2s cubic-bezier(1, 0.5, 0.8, 1);
}

.slide-fade-enter-from,
.slide-fade-leave-to {
  transform: translateX(20px);
  opacity: 0;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
```

### 6.2 HomeView.vue

```vue
<!-- src/views/HomeView.vue -->
<template>
  <div class="home-view">
    <!-- Welcome Section -->
    <section class="welcome-section">
      <div class="welcome-bg" />
      <div class="welcome-content">
        <div v-if="userStore.profile" class="user-greeting">
          <img
            :src="userStore.avatarUrl"
            :alt="userStore.displayName"
            class="avatar"
          />
          <div class="greeting-text">
            <p class="greeting-label">สวัสดี</p>
            <h1 class="greeting-name">{{ userStore.displayName }}</h1>
          </div>
        </div>

        <div v-else class="login-prompt">
          <div class="login-icon">👋</div>
          <h1>ยินดีต้อนรับ</h1>
          <p>เข้าสู่ระบบด้วย LINE เพื่อใช้งาน</p>
          <button @click="handleLogin" class="login-btn">
            <span class="line-icon" />
            เข้าสู่ระบบด้วย LINE
          </button>
        </div>

        <!-- Points Card -->
        <div v-if="userStore.profile" class="points-card">
          <div class="points-label">คะแนนสะสม</div>
          <div class="points-value">
            <span class="points-number">{{ formatPoints(userStore.totalPoints) }}</span>
            <span class="points-unit">คะแนน</span>
          </div>
          <div class="points-rank">ระดับ: {{ userStore.points?.rank || 'Bronze' }}</div>
        </div>
      </div>
    </section>

    <!-- Quick Actions -->
    <section class="actions-section">
      <h2 class="section-title">เมนูหลัก</h2>
      <div class="actions-grid">
        <button
          v-for="action in quickActions"
          :key="action.id"
          @click="action.handler"
          class="action-card"
          :class="{ disabled: action.requiresAuth && !liffStore.isAuthenticated }"
        >
          <span class="action-icon">{{ action.icon }}</span>
          <span class="action-label">{{ action.label }}</span>
        </button>
      </div>
    </section>

    <!-- Features Section -->
    <section class="features-section">
      <h2 class="section-title">ฟีเจอร์พิเศษ</h2>

      <!-- QR Scanner -->
      <div class="feature-card">
        <div class="feature-header">
          <span class="feature-icon">📱</span>
          <h3>สแกน QR Code</h3>
        </div>
        <p class="feature-desc">สแกน QR Code เพื่อรับแต้มหรือดูข้อมูลสินค้า</p>
        <button
          @click="handleScan"
          :disabled="!liffStore.canUseScanCode || scanner.isScanning.value"
          class="feature-btn"
        >
          {{ scanner.isScanning.value ? 'กำลังสแกน...' : 'สแกนเลย' }}
        </button>
        <div v-if="scanner.scanResult.value" class="scan-result">
          ผลลัพธ์: {{ scanner.scanResult.value }}
        </div>
      </div>

      <!-- Share Feature -->
      <div class="feature-card">
        <div class="feature-header">
          <span class="feature-icon">📤</span>
          <h3>แชร์ให้เพื่อน</h3>
        </div>
        <p class="feature-desc">แชร์โปรโมชันและข้อมูลน่าสนใจให้เพื่อน</p>
        <button
          @click="handleShare"
          :disabled="!liffStore.canUseShareTargetPicker || share.isSharing.value"
          class="feature-btn secondary"
        >
          แชร์โปรโมชัน
        </button>
      </div>
    </section>

    <!-- LIFF Info (Dev Mode) -->
    <section v-if="isDev" class="dev-info">
      <h3>📊 LIFF Info</h3>
      <pre>{{ liffInfo }}</pre>
    </section>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useLiffStore } from '@/stores/liff'
import { useUserStore } from '@/stores/user'
import { useLiffScanner } from '@/composables/useLiffScanner'
import { useLiffShare } from '@/composables/useLiffShare'

const router = useRouter()
const liffStore = useLiffStore()
const userStore = useUserStore()
const scanner = useLiffScanner()
const share = useLiffShare()

const isDev = import.meta.env.DEV

const liffInfo = computed(() => JSON.stringify({
  isInClient: liffStore.isInClient,
  os: liffStore.os,
  version: liffStore.version,
  language: liffStore.language,
}, null, 2))

const quickActions = [
  {
    id: 'profile',
    icon: '👤',
    label: 'โปรไฟล์',
    requiresAuth: true,
    handler: () => router.push('/profile'),
  },
  {
    id: 'points',
    icon: '⭐',
    label: 'คะแนน',
    requiresAuth: true,
    handler: () => router.push('/points'),
  },
  {
    id: 'scan',
    icon: '📷',
    label: 'สแกน',
    requiresAuth: true,
    handler: () => router.push('/scan'),
  },
  {
    id: 'share',
    icon: '📤',
    label: 'แชร์',
    requiresAuth: true,
    handler: () => router.push('/share'),
  },
]

const handleLogin = () => {
  liffStore.login(window.location.href)
}

const handleScan = async () => {
  const result = await scanner.scanQRCode()
  if (result) {
    // ประมวลผล QR code
    console.log('Scanned:', result)
  }
}

const handleShare = async () => {
  const message = share.createProductMessage({
    name: 'สินค้าพิเศษวันนี้',
    price: 299,
    imageUrl: 'https://example.com/product.jpg',
    url: 'https://example.com/product/1',
  })

  await share.shareToFriend([message])
}

const formatPoints = (points: number) => {
  return new Intl.NumberFormat('th-TH').format(points)
}

onMounted(async () => {
  if (liffStore.isAuthenticated && !userStore.profile) {
    await userStore.fetchProfile()
    await userStore.fetchPoints()
  }
})
</script>

<style lang="scss" scoped>
.home-view {
  min-height: 100vh;
  background: var(--bg-secondary);
}

.welcome-section {
  position: relative;
  padding: 24px 16px 40px;
  background: linear-gradient(135deg, #06c755 0%, #00a844 100%);
  overflow: hidden;

  .welcome-bg {
    position: absolute;
    inset: 0;
    background: url("data:image/svg+xml,...") no-repeat center;
    opacity: 0.1;
  }

  .welcome-content {
    position: relative;
    z-index: 1;
  }
}

.user-greeting {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 20px;

  .avatar {
    width: 60px;
    height: 60px;
    border-radius: 50%;
    border: 3px solid rgba(255, 255, 255, 0.8);
    object-fit: cover;
  }

  .greeting-label {
    color: rgba(255, 255, 255, 0.8);
    font-size: 14px;
    margin: 0;
  }

  .greeting-name {
    color: white;
    font-size: 22px;
    font-weight: 700;
    margin: 0;
  }
}

.login-prompt {
  text-align: center;
  color: white;

  .login-icon {
    font-size: 48px;
    margin-bottom: 12px;
  }

  h1 {
    font-size: 24px;
    margin-bottom: 8px;
  }

  p {
    opacity: 0.8;
    margin-bottom: 20px;
  }

  .login-btn {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 14px 28px;
    background: white;
    color: #06c755;
    border: none;
    border-radius: 30px;
    font-size: 16px;
    font-weight: 700;
    cursor: pointer;
  }
}

.points-card {
  background: rgba(255, 255, 255, 0.15);
  backdrop-filter: blur(10px);
  border-radius: 16px;
  padding: 20px;
  color: white;

  .points-label {
    font-size: 14px;
    opacity: 0.8;
    margin-bottom: 8px;
  }

  .points-value {
    display: flex;
    align-items: baseline;
    gap: 4px;
    margin-bottom: 4px;

    .points-number {
      font-size: 36px;
      font-weight: 700;
    }

    .points-unit {
      font-size: 16px;
    }
  }

  .points-rank {
    font-size: 14px;
    opacity: 0.8;
  }
}

.actions-section,
.features-section {
  padding: 24px 16px;

  .section-title {
    font-size: 18px;
    font-weight: 700;
    margin-bottom: 16px;
    color: var(--text-primary);
  }
}

.actions-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;

  .action-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    padding: 16px 8px;
    background: white;
    border: 1px solid var(--border-color);
    border-radius: 12px;
    cursor: pointer;

    &.disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }

    .action-icon {
      font-size: 28px;
    }

    .action-label {
      font-size: 12px;
      color: var(--text-secondary);
    }
  }
}

.feature-card {
  background: white;
  border-radius: 16px;
  padding: 20px;
  margin-bottom: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);

  .feature-header {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 8px;

    .feature-icon {
      font-size: 28px;
    }

    h3 {
      font-size: 16px;
      font-weight: 700;
    }
  }

  .feature-desc {
    color: var(--text-secondary);
    font-size: 14px;
    margin-bottom: 16px;
  }

  .feature-btn {
    width: 100%;
    padding: 12px;
    background: var(--primary-color);
    color: white;
    border: none;
    border-radius: 8px;
    font-size: 15px;
    cursor: pointer;

    &.secondary {
      background: var(--secondary-color);
    }

    &:disabled {
      opacity: 0.5;
      cursor: not-allowed;
    }
  }
}

.dev-info {
  margin: 16px;
  padding: 16px;
  background: #f8f9fa;
  border-radius: 8px;
  font-family: monospace;

  pre {
    font-size: 12px;
    overflow-x: auto;
  }
}
</style>
```

---

## 7. ProfileView.vue - หน้าโปรไฟล์

```vue
<!-- src/views/ProfileView.vue -->
<template>
  <div class="profile-view">
    <div v-if="userStore.isLoadingProfile" class="loading-state">
      <LoadingSpinner />
    </div>

    <template v-else-if="userStore.profile">
      <!-- Profile Header -->
      <div class="profile-header">
        <div class="avatar-container">
          <img
            :src="userStore.avatarUrl"
            :alt="userStore.displayName"
            class="avatar"
          />
          <div class="avatar-badge">✓</div>
        </div>

        <h1 class="display-name">{{ userStore.displayName }}</h1>
        <p v-if="userStore.profile.statusMessage" class="status-message">
          {{ userStore.profile.statusMessage }}
        </p>
        <p class="user-id">ID: {{ userStore.profile.userId }}</p>
      </div>

      <!-- Profile Details -->
      <div class="profile-details">
        <div class="detail-item" v-if="userStore.profile.email">
          <span class="detail-label">📧 อีเมล</span>
          <span class="detail-value">{{ userStore.profile.email }}</span>
        </div>

        <div class="detail-item">
          <span class="detail-label">⭐ คะแนนสะสม</span>
          <span class="detail-value points">{{ formatNumber(userStore.totalPoints) }}</span>
        </div>

        <div class="detail-item" v-if="userStore.points">
          <span class="detail-label">🏆 ระดับสมาชิก</span>
          <span class="detail-value rank" :class="userStore.points.rank.toLowerCase()">
            {{ userStore.points.rank }}
          </span>
        </div>
      </div>

      <!-- Action Buttons -->
      <div class="profile-actions">
        <button @click="handleShareProfile" class="action-btn">
          📤 แชร์โปรไฟล์
        </button>

        <button @click="handleSendToChat" class="action-btn secondary">
          💬 ส่งไปยังแชท
        </button>

        <button @click="handleLogout" class="action-btn danger">
          🚪 ออกจากระบบ
        </button>
      </div>
    </template>

    <div v-else class="empty-state">
      <p>ไม่พบข้อมูลผู้ใช้</p>
      <button @click="loadProfile" class="retry-btn">โหลดใหม่</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted } from 'vue'
import { useLiffStore } from '@/stores/liff'
import { useUserStore } from '@/stores/user'
import { useLiffShare } from '@/composables/useLiffShare'
import LoadingSpinner from '@/components/common/LoadingSpinner.vue'

const liffStore = useLiffStore()
const userStore = useUserStore()
const { shareToFriend, sendToChat } = useLiffShare()

const loadProfile = async () => {
  await userStore.fetchProfile()
  await userStore.fetchPoints()
}

const handleShareProfile = async () => {
  if (!userStore.profile) return

  await shareToFriend([
    {
      type: 'flex',
      altText: `โปรไฟล์ของ ${userStore.displayName}`,
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'image',
              url: userStore.avatarUrl,
              size: 'full',
              aspectRatio: '1:1',
              aspectMode: 'cover',
              action: { type: 'uri', uri: 'https://line.me' },
            },
            {
              type: 'text',
              text: userStore.displayName,
              weight: 'bold',
              size: 'xl',
              margin: 'md',
            },
          ],
        },
      },
    },
  ])
}

const handleSendToChat = async () => {
  if (!userStore.profile) return

  await sendToChat([
    {
      type: 'text',
      text: `สวัสดี! ฉันคือ ${userStore.displayName} มีคะแนนสะสม ${userStore.totalPoints} คะแนน 🎉`,
    },
  ])
}

const handleLogout = () => {
  userStore.clearUser()
  liffStore.logout()
}

const formatNumber = (num: number) => {
  return new Intl.NumberFormat('th-TH').format(num)
}

onMounted(async () => {
  if (!userStore.profile) {
    await loadProfile()
  }
})
</script>

<style lang="scss" scoped>
.profile-view {
  padding: 24px 16px;
}

.profile-header {
  text-align: center;
  margin-bottom: 32px;

  .avatar-container {
    position: relative;
    display: inline-block;
    margin-bottom: 16px;

    .avatar {
      width: 100px;
      height: 100px;
      border-radius: 50%;
      object-fit: cover;
      border: 4px solid #06c755;
    }

    .avatar-badge {
      position: absolute;
      bottom: 4px;
      right: 4px;
      width: 24px;
      height: 24px;
      background: #06c755;
      color: white;
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 12px;
      border: 2px solid white;
    }
  }

  .display-name {
    font-size: 24px;
    font-weight: 700;
    margin-bottom: 4px;
  }

  .status-message {
    color: var(--text-secondary);
    margin-bottom: 4px;
  }

  .user-id {
    font-size: 12px;
    color: var(--text-tertiary);
  }
}

.profile-details {
  background: white;
  border-radius: 16px;
  overflow: hidden;
  margin-bottom: 24px;

  .detail-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px;
    border-bottom: 1px solid var(--border-light);

    &:last-child {
      border-bottom: none;
    }

    .detail-label {
      color: var(--text-secondary);
      font-size: 15px;
    }

    .detail-value {
      font-weight: 600;

      &.points {
        color: #f5a623;
        font-size: 18px;
      }

      &.rank {
        padding: 2px 12px;
        border-radius: 12px;

        &.bronze {
          background: #cd7f32;
          color: white;
        }

        &.silver {
          background: #c0c0c0;
          color: white;
        }

        &.gold {
          background: #ffd700;
          color: white;
        }
      }
    }
  }
}

.profile-actions {
  display: flex;
  flex-direction: column;
  gap: 12px;

  .action-btn {
    width: 100%;
    padding: 14px;
    border: none;
    border-radius: 12px;
    font-size: 16px;
    cursor: pointer;
    background: #06c755;
    color: white;

    &.secondary {
      background: #3c87f4;
    }

    &.danger {
      background: #ff4444;
    }
  }
}
</style>
```

---

## 8. CSS Variables และ Global Styles

```scss
// src/assets/styles/variables.scss
:root {
  // Colors
  --primary-color: #06c755;
  --primary-dark: #00a844;
  --secondary-color: #3c87f4;
  --error-color: #ff4444;
  --warning-color: #f5a623;
  --success-color: #00b894;

  // Text
  --text-primary: #1a1a2e;
  --text-secondary: #6b7280;
  --text-tertiary: #9ca3af;

  // Background
  --bg-color: #ffffff;
  --bg-secondary: #f8f9fa;
  --bg-tertiary: #f1f3f4;

  // Border
  --border-color: #e5e7eb;
  --border-light: #f3f4f6;

  // Shadow
  --shadow-sm: 0 1px 3px rgba(0, 0, 0, 0.06);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.08);
  --shadow-lg: 0 8px 24px rgba(0, 0, 0, 0.12);

  // Spacing
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  // Border Radius
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
  --radius-full: 9999px;

  // Font
  --font-family: 'Sarabun', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

// Dark mode (LINE ใช้ dark mode ด้วย)
@media (prefers-color-scheme: dark) {
  :root {
    --text-primary: #f9fafb;
    --text-secondary: #9ca3af;
    --text-tertiary: #6b7280;
    --bg-color: #111827;
    --bg-secondary: #1f2937;
    --bg-tertiary: #374151;
    --border-color: #374151;
    --border-light: #1f2937;
  }
}
```

```scss
// src/assets/styles/main.scss
@import url('https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;500;600;700&display=swap');
@import './variables.scss';

*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html {
  font-size: 16px;
  -webkit-tap-highlight-color: transparent;
}

body {
  font-family: var(--font-family);
  background-color: var(--bg-secondary);
  color: var(--text-primary);
  line-height: 1.5;
  overscroll-behavior: none;
}

img {
  max-width: 100%;
  height: auto;
}

button {
  font-family: inherit;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;

  &:active {
    transform: scale(0.98);
  }
}

a {
  color: var(--primary-color);
  text-decoration: none;
}
```

---

## 9. Deployment

### 9.1 Deploy บน Vercel

```bash
# ติดตั้ง Vercel CLI
npm install -g vercel

# Build
npm run build

# Deploy
vercel --prod
```

**vercel.json:**
```json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": {
        "distDir": "dist"
      }
    }
  ],
  "routes": [
    {
      "src": "/(.*)",
      "dest": "/index.html"
    }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        }
      ]
    }
  ]
}
```

### 9.2 Deploy บน Netlify

**netlify.toml:**
```toml
[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[build.environment]
  NODE_VERSION = "18"
```

### 9.3 Deploy บน Firebase Hosting

```bash
# ติดตั้ง Firebase CLI
npm install -g firebase-tools

# Login
firebase login

# Init
firebase init hosting

# Build
npm run build

# Deploy
firebase deploy --only hosting
```

**firebase.json:**
```json
{
  "hosting": {
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [
          {
            "key": "Cache-Control",
            "value": "max-age=31536000"
          }
        ]
      }
    ]
  }
}
```

---

## 10. Testing

### 10.1 Unit Tests ด้วย Vitest

```typescript
// src/composables/__tests__/useLiffProfile.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { useLiffProfile } from '../useLiffProfile'

// Mock LIFF
vi.mock('@line/liff', () => ({
  default: {
    isLoggedIn: vi.fn(() => true),
    getProfile: vi.fn(() =>
      Promise.resolve({
        userId: 'U123456',
        displayName: 'Test User',
        pictureUrl: 'https://example.com/pic.jpg',
        statusMessage: 'Hello',
      })
    ),
    getIDToken: vi.fn(() => null),
  },
}))

describe('useLiffProfile', () => {
  it('should fetch profile successfully', async () => {
    const { profile, fetchProfile, isLoading } = useLiffProfile()

    expect(isLoading.value).toBe(false)

    await fetchProfile()

    expect(profile.value).toBeTruthy()
    expect(profile.value?.displayName).toBe('Test User')
    expect(profile.value?.userId).toBe('U123456')
    expect(isLoading.value).toBe(false)
  })

  it('should compute display name correctly', async () => {
    const { displayName, fetchProfile } = useLiffProfile()

    expect(displayName.value).toBe('ไม่ระบุชื่อ')

    await fetchProfile()

    expect(displayName.value).toBe('Test User')
  })
})
```

### 10.2 Component Tests ด้วย Vue Test Utils

```typescript
// src/views/__tests__/HomeView.test.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import { createPinia } from 'pinia'
import { createRouter, createMemoryHistory } from 'vue-router'
import HomeView from '../HomeView.vue'
import { useLiffStore } from '@/stores/liff'

const router = createRouter({
  history: createMemoryHistory(),
  routes: [{ path: '/', component: HomeView }],
})

describe('HomeView', () => {
  it('shows login button when not authenticated', async () => {
    const pinia = createPinia()

    const wrapper = mount(HomeView, {
      global: {
        plugins: [pinia, router],
      },
    })

    const liffStore = useLiffStore(pinia)
    liffStore.isInitialized = true
    liffStore.isLoggedIn = false

    await wrapper.vm.$nextTick()

    expect(wrapper.find('.login-btn').exists()).toBe(true)
  })
})
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. การตั้งค่า Vue 3 + Vite + LIFF project อย่างครบถ้วน
2. การสร้าง LIFF Plugin สำหรับ Vue ที่ reusable
3. Composables ต่างๆ ที่ครอบคลุม LIFF features ทั้งหมด
4. Pinia store สำหรับจัดการ LIFF state อย่างเป็นระบบ
5. Vue Router พร้อม Navigation Guard สำหรับ authentication
6. Complete UI components พร้อม styling
7. Deployment options หลายแบบ
8. Testing patterns สำหรับ LIFF composables และ components

การใช้ Vue 3 กับ LIFF ทำให้เราสามารถสร้าง Mini App ที่มีคุณภาพสูง มี code ที่ maintainable และมี developer experience ที่ดีมากขึ้นเมื่อเทียบกับการใช้ vanilla JavaScript
