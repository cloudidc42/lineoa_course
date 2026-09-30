# Part 86: Advanced LIFF Application Development

## สารบัญ

1. [LIFF Advanced Architecture](#liff-advanced-architecture)
2. [LIFF + PWA (Progressive Web App)](#liff-pwa)
3. [Offline Support with Service Workers](#offline-support)
4. [LIFF Performance Optimization](#performance-optimization)
5. [Code Splitting and Lazy Loading](#code-splitting)
6. [LIFF with Next.js](#liff-nextjs)
7. [Server-Side Rendering with LIFF](#ssr-liff)
8. [LIFF Debugging Tools](#debugging-tools)
9. [Analytics in LIFF](#analytics-liff)
10. [Complex LIFF App: Mini CRM](#mini-crm)
11. [Complete Next.js + LIFF Project](#complete-project)

---

## 1. LIFF Advanced Architecture {#liff-advanced-architecture}

### ภาพรวม LIFF Architecture ขั้นสูง

```
┌─────────────────────────────────────────────────────────────────┐
│                    LINE Platform                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐   │
│  │   LINE App   │    │  LIFF Server │    │   LINE Login     │   │
│  │  (WebView)   │◄──►│  (Channel)   │◄──►│   (OAuth 2.0)    │   │
│  └──────┬───────┘    └──────────────┘    └──────────────────┘   │
│         │                                                         │
└─────────┼───────────────────────────────────────────────────────┘
          │ HTTPS
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LIFF Application                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Next.js App Router                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │  │   Pages     │  │  Components │  │   API Routes     │  │   │
│  │  │  (RSC+CC)   │  │  (Client)   │  │   (Server)       │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Service      │  │    PWA       │  │   State Management   │   │
│  │ Worker       │  │  Manifest    │  │   (Zustand/Jotai)    │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │   │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Backend Services                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │  Node.js API │  │  PostgreSQL  │  │      Redis           │   │
│  │  (Fastify)   │  │  (Prisma)    │  │   (Cache/Session)    │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### โครงสร้างโปรเจค Advanced LIFF

```
liff-advanced-app/
├── apps/
│   ├── web/                    # Next.js LIFF App
│   │   ├── app/
│   │   │   ├── (liff)/         # LIFF Route Group
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   ├── profile/
│   │   │   │   ├── contacts/
│   │   │   │   └── crm/
│   │   │   ├── api/
│   │   │   │   ├── auth/
│   │   │   │   ├── users/
│   │   │   │   └── crm/
│   │   │   ├── globals.css
│   │   │   └── layout.tsx
│   │   ├── components/
│   │   │   ├── liff/
│   │   │   │   ├── LiffProvider.tsx
│   │   │   │   ├── LiffContext.tsx
│   │   │   │   └── LiffGuard.tsx
│   │   │   ├── ui/
│   │   │   └── features/
│   │   ├── lib/
│   │   │   ├── liff.ts
│   │   │   ├── auth.ts
│   │   │   └── api.ts
│   │   ├── hooks/
│   │   │   ├── useLiff.ts
│   │   │   ├── useProfile.ts
│   │   │   └── usePermissions.ts
│   │   ├── public/
│   │   │   ├── manifest.json
│   │   │   ├── sw.js
│   │   │   └── icons/
│   │   └── next.config.js
│   └── api/                    # Backend API
│       ├── src/
│       │   ├── routes/
│       │   ├── services/
│       │   ├── models/
│       │   └── middleware/
│       └── prisma/
├── packages/
│   ├── ui/                     # Shared UI Components
│   ├── liff-sdk/               # LIFF SDK Wrapper
│   └── types/                  # Shared TypeScript Types
├── docker/
├── turbo.json
└── package.json
```

---

## 2. LIFF + PWA (Progressive Web App) {#liff-pwa}

### Web App Manifest

```json
// public/manifest.json
{
  "name": "LINE Mini CRM",
  "short_name": "MiniCRM",
  "description": "ระบบ CRM สำหรับธุรกิจขนาดเล็กผ่าน LINE",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#00B900",
  "theme_color": "#00B900",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icons/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-96x96.png",
      "sizes": "96x96",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-128x128.png",
      "sizes": "128x128",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-144x144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-152x152.png",
      "sizes": "152x152",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-384x384.png",
      "sizes": "384x384",
      "type": "image/png",
      "purpose": "maskable any"
    },
    {
      "src": "/icons/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable any"
    }
  ],
  "screenshots": [
    {
      "src": "/screenshots/mobile-1.png",
      "type": "image/png",
      "sizes": "390x844",
      "form_factor": "narrow"
    }
  ],
  "categories": ["business", "productivity"],
  "shortcuts": [
    {
      "name": "เพิ่มลูกค้า",
      "short_name": "เพิ่มลูกค้า",
      "description": "เพิ่มลูกค้าใหม่เข้าระบบ",
      "url": "/crm/contacts/new",
      "icons": [{"src": "/icons/add-contact.png", "sizes": "192x192"}]
    },
    {
      "name": "รายงานวันนี้",
      "short_name": "รายงาน",
      "description": "ดูรายงานประจำวัน",
      "url": "/crm/reports/today",
      "icons": [{"src": "/icons/report.png", "sizes": "192x192"}]
    }
  ]
}
```

### Next.js Configuration สำหรับ PWA

```javascript
// next.config.js
const withPWA = require('next-pwa')({
  dest: 'public',
  register: true,
  skipWaiting: true,
  disable: process.env.NODE_ENV === 'development',
  runtimeCaching: [
    {
      urlPattern: /^https:\/\/api\.line\.me\/.*/i,
      handler: 'NetworkFirst',
      options: {
        cacheName: 'line-api-cache',
        expiration: {
          maxEntries: 32,
          maxAgeSeconds: 24 * 60 * 60 // 24 hours
        }
      }
    },
    {
      urlPattern: /^https:\/\/profile\.line-scdn\.net\/.*/i,
      handler: 'CacheFirst',
      options: {
        cacheName: 'line-profile-images',
        expiration: {
          maxEntries: 64,
          maxAgeSeconds: 7 * 24 * 60 * 60 // 7 days
        }
      }
    },
    {
      urlPattern: /\/api\/crm\/.*/i,
      handler: 'NetworkFirst',
      options: {
        cacheName: 'crm-api-cache',
        expiration: {
          maxEntries: 100,
          maxAgeSeconds: 5 * 60 // 5 minutes
        },
        networkTimeoutSeconds: 10
      }
    },
    {
      urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp)$/i,
      handler: 'CacheFirst',
      options: {
        cacheName: 'static-images',
        expiration: {
          maxEntries: 64,
          maxAgeSeconds: 30 * 24 * 60 * 60 // 30 days
        }
      }
    }
  ]
})

/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  swcMinify: true,
  images: {
    domains: [
      'profile.line-scdn.net',
      'obs.line-scdn.net'
    ]
  },
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        {
          key: 'X-Frame-Options',
          value: 'SAMEORIGIN'
        },
        {
          key: 'X-Content-Type-Options',
          value: 'nosniff'
        },
        {
          key: 'Referrer-Policy',
          value: 'strict-origin-when-cross-origin'
        }
      ]
    }
  ],
  experimental: {
    optimizeCss: true,
    optimizeServerReact: true
  }
}

module.exports = withPWA(nextConfig)
```

---

## 3. Offline Support with Service Workers {#offline-support}

### Custom Service Worker

```javascript
// public/sw.js
import { precacheAndRoute, cleanupOutdatedCaches } from 'workbox-precaching'
import { registerRoute, NavigationRoute } from 'workbox-routing'
import { 
  NetworkFirst, 
  CacheFirst, 
  StaleWhileRevalidate,
  NetworkOnly
} from 'workbox-strategies'
import { BackgroundSyncPlugin } from 'workbox-background-sync'
import { ExpirationPlugin } from 'workbox-expiration'
import { CacheableResponsePlugin } from 'workbox-cacheable-response'

// Precache static assets
precacheAndRoute(self.__WB_MANIFEST)
cleanupOutdatedCaches()

// Background Sync สำหรับ offline mutations
const bgSyncPlugin = new BackgroundSyncPlugin('crm-mutations-queue', {
  maxRetentionTime: 24 * 60, // 24 hours
  onSync: async ({ queue }) => {
    let entry
    while ((entry = await queue.shiftRequest())) {
      try {
        await fetch(entry.request)
        console.log('Background sync: request replayed', entry.request.url)
        
        // แจ้งเตือน client ว่า sync สำเร็จ
        const clients = await self.clients.matchAll()
        clients.forEach(client => {
          client.postMessage({
            type: 'BACKGROUND_SYNC_SUCCESS',
            url: entry.request.url
          })
        })
      } catch (error) {
        console.error('Background sync: replay failed', error)
        await queue.unshiftRequest(entry)
        throw error
      }
    }
  }
})

// CRM API routes - NetworkFirst with background sync
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/crm/contacts'),
  new NetworkFirst({
    cacheName: 'crm-contacts-cache',
    plugins: [
      new CacheableResponsePlugin({ statuses: [200] }),
      new ExpirationPlugin({ maxEntries: 200, maxAgeSeconds: 10 * 60 })
    ]
  }),
  'GET'
)

// POST/PUT/DELETE - Background Sync
registerRoute(
  ({ url, request }) => 
    url.pathname.startsWith('/api/crm/') && 
    ['POST', 'PUT', 'DELETE'].includes(request.method),
  new NetworkOnly({
    plugins: [bgSyncPlugin]
  }),
  'POST'
)

// LINE API - StaleWhileRevalidate
registerRoute(
  ({ url }) => url.hostname === 'api.line.me',
  new StaleWhileRevalidate({
    cacheName: 'line-api-cache',
    plugins: [
      new ExpirationPlugin({ maxEntries: 50, maxAgeSeconds: 60 * 60 })
    ]
  })
)

// Offline fallback page
const offlineFallbackPage = '/offline'
registerRoute(
  new NavigationRoute(
    new NetworkFirst({
      cacheName: 'pages-cache',
      plugins: [
        new CacheableResponsePlugin({ statuses: [200] }),
        new ExpirationPlugin({ maxEntries: 50 })
      ]
    }),
    { denylist: [/^\/api\//] }
  )
)

// Handle skip waiting
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting()
  }
})

// Push notifications
self.addEventListener('push', (event) => {
  if (!event.data) return

  const data = event.data.json()
  
  const options = {
    body: data.body,
    icon: '/icons/icon-192x192.png',
    badge: '/icons/badge-72x72.png',
    vibrate: [100, 50, 100],
    data: {
      url: data.url,
      dateOfArrival: Date.now(),
      primaryKey: data.id
    },
    actions: [
      { action: 'explore', title: 'ดูรายละเอียด', icon: '/icons/checkmark.png' },
      { action: 'close', title: 'ปิด', icon: '/icons/xmark.png' }
    ]
  }

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  )
})

// Notification click
self.addEventListener('notificationclick', (event) => {
  event.notification.close()

  if (event.action === 'explore') {
    event.waitUntil(
      clients.openWindow(event.notification.data.url)
    )
  }
})
```

### IndexedDB สำหรับ Offline Storage

```typescript
// lib/offline-db.ts
import { openDB, DBSchema, IDBPDatabase } from 'idb'

interface CRMDatabase extends DBSchema {
  contacts: {
    key: string
    value: Contact
    indexes: {
      'by-name': string
      'by-updated': Date
      'by-status': string
    }
  }
  pendingMutations: {
    key: number
    value: PendingMutation
    autoIncrement: true
  }
  syncLog: {
    key: number
    value: SyncLogEntry
    autoIncrement: true
  }
}

interface Contact {
  id: string
  lineUserId: string
  displayName: string
  email?: string
  phone?: string
  tags: string[]
  status: 'active' | 'inactive' | 'prospect'
  notes: string
  createdAt: Date
  updatedAt: Date
  _syncStatus?: 'synced' | 'pending' | 'conflict'
}

interface PendingMutation {
  id?: number
  type: 'create' | 'update' | 'delete'
  entity: 'contact' | 'deal' | 'activity'
  entityId: string
  payload: unknown
  timestamp: Date
  retryCount: number
}

interface SyncLogEntry {
  id?: number
  timestamp: Date
  action: string
  status: 'success' | 'failed'
  details: string
}

class OfflineDatabase {
  private db: IDBPDatabase<CRMDatabase> | null = null

  async init(): Promise<void> {
    this.db = await openDB<CRMDatabase>('mini-crm', 2, {
      upgrade(db, oldVersion) {
        // Version 1
        if (oldVersion < 1) {
          const contactStore = db.createObjectStore('contacts', { keyPath: 'id' })
          contactStore.createIndex('by-name', 'displayName')
          contactStore.createIndex('by-updated', 'updatedAt')
          contactStore.createIndex('by-status', 'status')

          db.createObjectStore('pendingMutations', {
            keyPath: 'id',
            autoIncrement: true
          })
        }

        // Version 2
        if (oldVersion < 2) {
          db.createObjectStore('syncLog', {
            keyPath: 'id',
            autoIncrement: true
          })
        }
      }
    })
  }

  private getDB(): IDBPDatabase<CRMDatabase> {
    if (!this.db) throw new Error('Database not initialized')
    return this.db
  }

  // Contacts CRUD
  async getContact(id: string): Promise<Contact | undefined> {
    return this.getDB().get('contacts', id)
  }

  async getAllContacts(): Promise<Contact[]> {
    return this.getDB().getAll('contacts')
  }

  async getContactsByStatus(status: Contact['status']): Promise<Contact[]> {
    return this.getDB().getAllFromIndex('contacts', 'by-status', status)
  }

  async upsertContact(contact: Contact): Promise<void> {
    await this.getDB().put('contacts', contact)
  }

  async deleteContact(id: string): Promise<void> {
    await this.getDB().delete('contacts', id)
  }

  async searchContacts(query: string): Promise<Contact[]> {
    const all = await this.getAllContacts()
    const lower = query.toLowerCase()
    return all.filter(c =>
      c.displayName.toLowerCase().includes(lower) ||
      c.email?.toLowerCase().includes(lower) ||
      c.phone?.includes(query)
    )
  }

  // Pending mutations
  async addPendingMutation(mutation: Omit<PendingMutation, 'id'>): Promise<void> {
    await this.getDB().add('pendingMutations', mutation)
  }

  async getPendingMutations(): Promise<PendingMutation[]> {
    return this.getDB().getAll('pendingMutations')
  }

  async deletePendingMutation(id: number): Promise<void> {
    await this.getDB().delete('pendingMutations', id)
  }

  // Sync
  async syncWithServer(): Promise<{ synced: number; failed: number }> {
    const pending = await this.getPendingMutations()
    let synced = 0
    let failed = 0

    for (const mutation of pending) {
      try {
        await this.applyMutation(mutation)
        await this.deletePendingMutation(mutation.id!)
        synced++
      } catch (error) {
        failed++
        await this.getDB().put('pendingMutations', {
          ...mutation,
          retryCount: mutation.retryCount + 1
        })
      }
    }

    await this.logSync(`Synced ${synced} mutations, ${failed} failed`)
    return { synced, failed }
  }

  private async applyMutation(mutation: PendingMutation): Promise<void> {
    const url = `/api/crm/${mutation.entity}s/${mutation.entityId}`
    const method = mutation.type === 'create' ? 'POST' :
                   mutation.type === 'update' ? 'PUT' : 'DELETE'

    const response = await fetch(url, {
      method,
      headers: { 'Content-Type': 'application/json' },
      body: mutation.type !== 'delete' ? JSON.stringify(mutation.payload) : undefined
    })

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`)
    }
  }

  private async logSync(details: string): Promise<void> {
    await this.getDB().add('syncLog', {
      timestamp: new Date(),
      action: 'sync',
      status: 'success',
      details
    })
  }
}

export const offlineDB = new OfflineDatabase()
```

---

## 4. LIFF Performance Optimization {#performance-optimization}

### Resource Hints และ Preloading

```typescript
// app/layout.tsx
import { Metadata, Viewport } from 'next'

export const metadata: Metadata = {
  title: 'LINE Mini CRM',
  description: 'ระบบ CRM ผ่าน LINE OA',
  manifest: '/manifest.json',
  appleWebApp: {
    capable: true,
    statusBarStyle: 'default',
    title: 'Mini CRM'
  },
  formatDetection: {
    telephone: false
  }
}

export const viewport: Viewport = {
  themeColor: '#00B900',
  width: 'device-width',
  initialScale: 1,
  maximumScale: 1,
  userScalable: false
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="th">
      <head>
        {/* DNS Prefetch */}
        <link rel="dns-prefetch" href="https://api.line.me" />
        <link rel="dns-prefetch" href="https://profile.line-scdn.net" />
        <link rel="dns-prefetch" href="https://access.line.me" />
        
        {/* Preconnect */}
        <link rel="preconnect" href="https://api.line.me" crossOrigin="anonymous" />
        <link rel="preconnect" href="https://static.line-scdn.net" crossOrigin="anonymous" />
        
        {/* Preload LIFF SDK */}
        <link 
          rel="preload" 
          href="https://static.line-scdn.net/liff/edge/2/sdk.js"
          as="script"
          crossOrigin="anonymous"
        />
        
        {/* Apple Touch Icons */}
        <link rel="apple-touch-icon" href="/icons/icon-192x192.png" />
      </head>
      <body>
        {children}
      </body>
    </html>
  )
}
```

### LIFF SDK Wrapper ที่ Optimized

```typescript
// lib/liff.ts
import type { Liff } from '@line/liff'

type LiffStatus = 'idle' | 'loading' | 'ready' | 'error'

interface LiffState {
  liff: Liff | null
  status: LiffStatus
  error: Error | null
  isInClient: boolean
  isLoggedIn: boolean
  profile: LiffProfile | null
}

interface LiffProfile {
  userId: string
  displayName: string
  pictureUrl?: string
  statusMessage?: string
}

// Singleton LIFF instance
let liffInstance: Liff | null = null
let initPromise: Promise<Liff> | null = null

export async function initLiff(liffId: string): Promise<Liff> {
  // Return existing instance
  if (liffInstance) return liffInstance
  
  // Return in-flight promise
  if (initPromise) return initPromise

  initPromise = new Promise(async (resolve, reject) => {
    try {
      // Dynamic import เพื่อลด initial bundle size
      const { default: liff } = await import('@line/liff')
      
      await liff.init({
        liffId,
        withLoginOnExternalBrowser: true
      })

      liffInstance = liff
      resolve(liff)
    } catch (error) {
      initPromise = null
      reject(error)
    }
  })

  return initPromise
}

export function getLiff(): Liff | null {
  return liffInstance
}

// Profile cache
const profileCache = new Map<string, { profile: LiffProfile; timestamp: number }>()
const PROFILE_CACHE_TTL = 5 * 60 * 1000 // 5 minutes

export async function getProfile(liff: Liff): Promise<LiffProfile> {
  const cacheKey = 'current-user'
  const cached = profileCache.get(cacheKey)
  
  if (cached && Date.now() - cached.timestamp < PROFILE_CACHE_TTL) {
    return cached.profile
  }

  const profile = await liff.getProfile()
  const mappedProfile: LiffProfile = {
    userId: profile.userId,
    displayName: profile.displayName,
    pictureUrl: profile.pictureUrl,
    statusMessage: profile.statusMessage
  }

  profileCache.set(cacheKey, {
    profile: mappedProfile,
    timestamp: Date.now()
  })

  return mappedProfile
}

export function clearProfileCache(): void {
  profileCache.clear()
}

// Token management
let accessToken: string | null = null
let tokenExpiry: number | null = null

export function getAccessToken(liff: Liff): string | null {
  if (tokenExpiry && Date.now() > tokenExpiry) {
    accessToken = null
    tokenExpiry = null
  }

  if (!accessToken) {
    accessToken = liff.getAccessToken()
    // LINE access token expires in 1 hour
    tokenExpiry = Date.now() + 55 * 60 * 1000
  }

  return accessToken
}
```

### LIFF Provider ที่ Optimized

```typescript
// components/liff/LiffProvider.tsx
'use client'

import React, { createContext, useContext, useEffect, useReducer, useCallback } from 'react'
import type { Liff } from '@line/liff'
import { initLiff, getProfile } from '@/lib/liff'

interface LiffContextValue {
  liff: Liff | null
  isReady: boolean
  isLoggedIn: boolean
  isInClient: boolean
  profile: LiffProfile | null
  error: Error | null
  login: () => void
  logout: () => void
  sendMessage: (message: LiffMessage) => Promise<void>
  closeWindow: () => void
}

type LiffAction =
  | { type: 'INIT_START' }
  | { type: 'INIT_SUCCESS'; payload: { liff: Liff; isInClient: boolean; isLoggedIn: boolean } }
  | { type: 'INIT_ERROR'; payload: Error }
  | { type: 'PROFILE_LOADED'; payload: LiffProfile }
  | { type: 'LOGOUT' }

interface LiffState {
  liff: Liff | null
  isReady: boolean
  isLoggedIn: boolean
  isInClient: boolean
  profile: LiffProfile | null
  error: Error | null
}

function liffReducer(state: LiffState, action: LiffAction): LiffState {
  switch (action.type) {
    case 'INIT_START':
      return { ...state, error: null }
    case 'INIT_SUCCESS':
      return {
        ...state,
        liff: action.payload.liff,
        isReady: true,
        isInClient: action.payload.isInClient,
        isLoggedIn: action.payload.isLoggedIn
      }
    case 'INIT_ERROR':
      return { ...state, error: action.payload, isReady: true }
    case 'PROFILE_LOADED':
      return { ...state, profile: action.payload }
    case 'LOGOUT':
      return { ...state, isLoggedIn: false, profile: null }
    default:
      return state
  }
}

const LiffContext = createContext<LiffContextValue | null>(null)

interface LiffProviderProps {
  liffId: string
  children: React.ReactNode
  fallback?: React.ReactNode
  onReady?: (liff: Liff) => void
}

export function LiffProvider({ liffId, children, fallback, onReady }: LiffProviderProps) {
  const [state, dispatch] = useReducer(liffReducer, {
    liff: null,
    isReady: false,
    isLoggedIn: false,
    isInClient: false,
    profile: null,
    error: null
  })

  useEffect(() => {
    let mounted = true

    async function initialize() {
      dispatch({ type: 'INIT_START' })
      
      try {
        const liff = await initLiff(liffId)
        
        if (!mounted) return

        dispatch({
          type: 'INIT_SUCCESS',
          payload: {
            liff,
            isInClient: liff.isInClient(),
            isLoggedIn: liff.isLoggedIn()
          }
        })

        if (liff.isLoggedIn()) {
          try {
            const profile = await getProfile(liff)
            if (mounted) {
              dispatch({ type: 'PROFILE_LOADED', payload: profile })
            }
          } catch (profileError) {
            console.warn('Failed to load profile:', profileError)
          }
        }

        onReady?.(liff)
      } catch (error) {
        if (mounted) {
          dispatch({ type: 'INIT_ERROR', payload: error as Error })
        }
      }
    }

    initialize()
    return () => { mounted = false }
  }, [liffId])

  const login = useCallback(() => {
    state.liff?.login({ redirectUri: window.location.href })
  }, [state.liff])

  const logout = useCallback(() => {
    state.liff?.logout()
    dispatch({ type: 'LOGOUT' })
  }, [state.liff])

  const sendMessage = useCallback(async (message: LiffMessage) => {
    if (!state.liff) throw new Error('LIFF not initialized')
    await state.liff.sendMessages([message])
  }, [state.liff])

  const closeWindow = useCallback(() => {
    state.liff?.closeWindow()
  }, [state.liff])

  const value: LiffContextValue = {
    ...state,
    login,
    logout,
    sendMessage,
    closeWindow
  }

  if (!state.isReady) {
    return fallback ?? <LiffLoadingScreen />
  }

  return (
    <LiffContext.Provider value={value}>
      {children}
    </LiffContext.Provider>
  )
}

export function useLiff(): LiffContextValue {
  const context = useContext(LiffContext)
  if (!context) {
    throw new Error('useLiff must be used within LiffProvider')
  }
  return context
}

function LiffLoadingScreen() {
  return (
    <div className="flex items-center justify-center min-h-screen bg-white">
      <div className="text-center">
        <div className="w-16 h-16 border-4 border-green-500 border-t-transparent rounded-full animate-spin mx-auto mb-4" />
        <p className="text-gray-600">กำลังโหลด LINE Mini CRM...</p>
      </div>
    </div>
  )
}
```

---

## 5. Code Splitting and Lazy Loading {#code-splitting}

### Dynamic Imports กับ Next.js

```typescript
// app/(liff)/crm/page.tsx
import dynamic from 'next/dynamic'
import { Suspense } from 'react'

// Lazy load components ที่ไม่จำเป็นใน first render
const ContactList = dynamic(
  () => import('@/components/features/crm/ContactList'),
  {
    loading: () => <ContactListSkeleton />,
    ssr: false
  }
)

const DealPipeline = dynamic(
  () => import('@/components/features/crm/DealPipeline'),
  {
    loading: () => <DealPipelineSkeleton />,
    ssr: false
  }
)

const ActivityFeed = dynamic(
  () => import('@/components/features/crm/ActivityFeed'),
  {
    loading: () => <ActivityFeedSkeleton />,
    ssr: false
  }
)

// Chart library (heavy) - load only when needed
const AnalyticsChart = dynamic(
  () => import('@/components/features/analytics/AnalyticsChart'),
  {
    loading: () => <div className="h-64 bg-gray-100 animate-pulse rounded-lg" />,
    ssr: false
  }
)

export default function CRMDashboard() {
  return (
    <div className="p-4 space-y-4">
      <h1 className="text-xl font-bold text-gray-900">Dashboard</h1>
      
      <Suspense fallback={<ContactListSkeleton />}>
        <ContactList />
      </Suspense>
      
      <Suspense fallback={<DealPipelineSkeleton />}>
        <DealPipeline />
      </Suspense>
      
      <Suspense fallback={<div className="h-64 bg-gray-100 animate-pulse rounded-lg" />}>
        <AnalyticsChart />
      </Suspense>
    </div>
  )
}
```

### Bundle Analysis

```bash
# วิเคราะห์ bundle size
npm install --save-dev @next/bundle-analyzer

# next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true'
})

module.exports = withBundleAnalyzer(nextConfig)

# รัน analysis
ANALYZE=true npm run build
```

### Code Splitting Strategy

```typescript
// hooks/useFeatureFlags.ts
'use client'

import { useState, useEffect } from 'react'

interface FeatureFlags {
  enableAnalytics: boolean
  enableDealPipeline: boolean
  enableAIAssistant: boolean
  enableBulkImport: boolean
}

const DEFAULT_FLAGS: FeatureFlags = {
  enableAnalytics: false,
  enableDealPipeline: false,
  enableAIAssistant: false,
  enableBulkImport: false
}

export function useFeatureFlags(): FeatureFlags {
  const [flags, setFlags] = useState<FeatureFlags>(DEFAULT_FLAGS)

  useEffect(() => {
    // โหลด feature flags จาก API
    fetch('/api/feature-flags')
      .then(r => r.json())
      .then(setFlags)
      .catch(() => setFlags(DEFAULT_FLAGS))
  }, [])

  return flags
}

// การใช้งาน: โหลด component เฉพาะเมื่อ feature enabled
export function ConditionalFeature({ 
  flag, 
  children 
}: { 
  flag: keyof FeatureFlags
  children: React.ReactNode 
}) {
  const flags = useFeatureFlags()
  
  if (!flags[flag]) return null
  
  return <>{children}</>
}
```

---

## 6. LIFF with Next.js {#liff-nextjs}

### App Router Layout

```typescript
// app/(liff)/layout.tsx
'use client'

import { LiffProvider } from '@/components/liff/LiffProvider'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useState } from 'react'

const LIFF_ID = process.env.NEXT_PUBLIC_LIFF_ID!

export default function LiffLayout({
  children,
}: {
  children: React.ReactNode
}) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minute
        gcTime: 5 * 60 * 1000, // 5 minutes
        retry: (failureCount, error) => {
          // ไม่ retry ถ้า 4xx
          if (error instanceof Error && error.message.includes('4')) return false
          return failureCount < 3
        }
      }
    }
  }))

  return (
    <QueryClientProvider client={queryClient}>
      <LiffProvider
        liffId={LIFF_ID}
        fallback={
          <div className="flex items-center justify-center min-h-screen">
            <LoadingSpinner />
          </div>
        }
      >
        <div className="min-h-screen bg-gray-50 max-w-md mx-auto">
          {children}
        </div>
      </LiffProvider>
    </QueryClientProvider>
  )
}
```

### Authentication Middleware

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { verifyLiffToken } from '@/lib/auth'

const PROTECTED_ROUTES = ['/api/crm', '/api/users']
const LIFF_ROUTES = ['/crm', '/contacts', '/deals']

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl

  // ตรวจสอบ LIFF routes
  if (LIFF_ROUTES.some(route => pathname.startsWith(route))) {
    const liffState = request.cookies.get('liff_state')
    
    if (!liffState) {
      return NextResponse.redirect(new URL('/', request.url))
    }
  }

  // ตรวจสอบ API routes
  if (PROTECTED_ROUTES.some(route => pathname.startsWith(route))) {
    const authHeader = request.headers.get('authorization')
    
    if (!authHeader?.startsWith('Bearer ')) {
      return NextResponse.json(
        { error: 'Unauthorized', message: 'Missing access token' },
        { status: 401 }
      )
    }

    const token = authHeader.slice(7)
    
    try {
      const userId = await verifyLiffToken(token)
      
      // ส่ง userId ผ่าน header
      const requestHeaders = new Headers(request.headers)
      requestHeaders.set('x-line-user-id', userId)
      
      return NextResponse.next({
        request: { headers: requestHeaders }
      })
    } catch (error) {
      return NextResponse.json(
        { error: 'Unauthorized', message: 'Invalid token' },
        { status: 401 }
      )
    }
  }

  return NextResponse.next()
}

export const config = {
  matcher: [
    '/api/crm/:path*',
    '/api/users/:path*',
    '/crm/:path*',
    '/contacts/:path*',
    '/deals/:path*'
  ]
}
```

---

## 7. Server-Side Rendering with LIFF {#ssr-liff}

### LIFF SSR Strategy

```
ปัญหา: LIFF SDK ต้องการ browser APIs (window, document)
       แต่ Next.js SSR รัน code บน server

วิธีแก้:
1. ใช้ 'use client' directive สำหรับ LIFF components
2. Dynamic import ด้วย ssr: false
3. แยก server/client concerns ชัดเจน
```

```typescript
// app/(liff)/contacts/[id]/page.tsx
// Server Component - fetch data บน server
import { Suspense } from 'react'
import { getContactById } from '@/lib/server/crm'
import ContactDetailClient from './ContactDetailClient'

interface Props {
  params: { id: string }
}

export default async function ContactDetailPage({ params }: Props) {
  // Fetch initial data บน server
  const contact = await getContactById(params.id)
  
  if (!contact) {
    return <div>ไม่พบข้อมูลลูกค้า</div>
  }

  return (
    <div>
      {/* Server rendered content */}
      <h1 className="text-xl font-bold p-4">{contact.displayName}</h1>
      
      {/* Client component สำหรับ interactive features */}
      <Suspense fallback={<ContactDetailSkeleton />}>
        <ContactDetailClient 
          initialData={contact}
          contactId={params.id}
        />
      </Suspense>
    </div>
  )
}

// Generate metadata
export async function generateMetadata({ params }: Props) {
  const contact = await getContactById(params.id)
  
  return {
    title: contact ? `${contact.displayName} - Mini CRM` : 'ไม่พบข้อมูล',
    description: contact?.notes?.slice(0, 160)
  }
}
```

```typescript
// app/(liff)/contacts/[id]/ContactDetailClient.tsx
'use client'

import { useLiff } from '@/components/liff/LiffProvider'
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { useState } from 'react'

interface Props {
  initialData: Contact
  contactId: string
}

export default function ContactDetailClient({ initialData, contactId }: Props) {
  const { liff, isLoggedIn, profile } = useLiff()
  const queryClient = useQueryClient()
  const [isEditing, setIsEditing] = useState(false)

  // ใช้ initialData เป็น initial cache
  const { data: contact } = useQuery({
    queryKey: ['contact', contactId],
    queryFn: () => fetch(`/api/crm/contacts/${contactId}`).then(r => r.json()),
    initialData,
    staleTime: 30 * 1000
  })

  const updateMutation = useMutation({
    mutationFn: (data: Partial<Contact>) =>
      fetch(`/api/crm/contacts/${contactId}`, {
        method: 'PUT',
        headers: { 
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${liff?.getAccessToken()}`
        },
        body: JSON.stringify(data)
      }).then(r => r.json()),
    onSuccess: (updated) => {
      queryClient.setQueryData(['contact', contactId], updated)
      setIsEditing(false)
    }
  })

  const sendLineMessage = async () => {
    if (!liff?.isInClient()) {
      alert('ฟีเจอร์นี้ใช้ได้เฉพาะใน LINE เท่านั้น')
      return
    }

    await liff.sendMessages([{
      type: 'flex',
      altText: `ข้อมูลลูกค้า: ${contact.displayName}`,
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: contact.displayName, weight: 'bold', size: 'xl' },
            { type: 'text', text: contact.email ?? 'ไม่มีอีเมล', color: '#666666' },
            { type: 'text', text: contact.phone ?? 'ไม่มีเบอร์โทร', color: '#666666' }
          ]
        }
      }
    }])
  }

  return (
    <div className="p-4 space-y-4">
      {/* Contact details */}
      {isEditing ? (
        <ContactEditForm 
          contact={contact}
          onSave={(data) => updateMutation.mutate(data)}
          onCancel={() => setIsEditing(false)}
        />
      ) : (
        <ContactView 
          contact={contact}
          onEdit={() => setIsEditing(true)}
          onShare={sendLineMessage}
        />
      )}
    </div>
  )
}
```

---

## 8. LIFF Debugging Tools {#debugging-tools}

### Debug Panel Component

```typescript
// components/debug/LiffDebugPanel.tsx
'use client'

import { useState, useEffect } from 'react'
import { useLiff } from '@/components/liff/LiffProvider'

export function LiffDebugPanel() {
  const { liff, isReady, isLoggedIn, isInClient, profile, error } = useLiff()
  const [isOpen, setIsOpen] = useState(false)
  const [liffInfo, setLiffInfo] = useState<Record<string, unknown>>({})

  // แสดงเฉพาะ development
  if (process.env.NODE_ENV !== 'development') return null

  useEffect(() => {
    if (!liff || !isReady) return

    const info: Record<string, unknown> = {
      liffId: process.env.NEXT_PUBLIC_LIFF_ID,
      version: liff.getVersion(),
      os: liff.getOS(),
      language: liff.getLanguage(),
      lineVersion: liff.getLineVersion(),
      isInClient: liff.isInClient(),
      isLoggedIn: liff.isLoggedIn(),
      isExpired: liff.isExpired(),
      context: liff.getContext(),
    }

    if (liff.isLoggedIn()) {
      info.accessToken = liff.getAccessToken()?.slice(0, 20) + '...'
      info.decodedToken = liff.getDecodedIDToken()
    }

    setLiffInfo(info)
  }, [liff, isReady])

  return (
    <div className="fixed bottom-4 right-4 z-50">
      <button
        onClick={() => setIsOpen(!isOpen)}
        className="bg-green-500 text-white px-3 py-2 rounded-full text-sm font-bold shadow-lg"
      >
        🐛 DEBUG
      </button>
      
      {isOpen && (
        <div className="absolute bottom-12 right-0 w-80 bg-gray-900 text-green-400 rounded-lg p-4 font-mono text-xs overflow-auto max-h-96 shadow-xl">
          <div className="flex justify-between items-center mb-2">
            <span className="text-white font-bold">LIFF Debug Info</span>
            <button 
              onClick={() => setIsOpen(false)}
              className="text-gray-400 hover:text-white"
            >
              ✕
            </button>
          </div>
          
          {error && (
            <div className="bg-red-900 text-red-300 p-2 rounded mb-2">
              <strong>ERROR:</strong> {error.message}
            </div>
          )}
          
          <pre className="whitespace-pre-wrap break-all">
            {JSON.stringify(liffInfo, null, 2)}
          </pre>
          
          <div className="mt-3 space-y-1 border-t border-gray-700 pt-3">
            <button
              onClick={() => {
                const token = liff?.getAccessToken()
                if (token) {
                  navigator.clipboard.writeText(token)
                  alert('Copied access token!')
                }
              }}
              className="w-full bg-gray-700 hover:bg-gray-600 text-white py-1 px-2 rounded text-xs"
            >
              Copy Access Token
            </button>
            
            <button
              onClick={async () => {
                const profile = await liff?.getProfile()
                console.log('Profile:', profile)
              }}
              className="w-full bg-gray-700 hover:bg-gray-600 text-white py-1 px-2 rounded text-xs"
            >
              Log Profile to Console
            </button>

            <button
              onClick={() => {
                const context = liff?.getContext()
                console.log('LIFF Context:', context)
              }}
              className="w-full bg-gray-700 hover:bg-gray-600 text-white py-1 px-2 rounded text-xs"
            >
              Log Context to Console
            </button>
          </div>
        </div>
      )}
    </div>
  )
}
```

### LIFF Mock สำหรับ Testing

```typescript
// __mocks__/@line/liff.ts
// Mock LIFF SDK สำหรับ unit tests

const mockProfile = {
  userId: 'U1234567890abcdef',
  displayName: 'Test User',
  pictureUrl: 'https://example.com/profile.jpg',
  statusMessage: 'Testing...'
}

const mockContext = {
  type: 'chat',
  viewType: 'full',
  userId: mockProfile.userId,
  utouId: 'U1234567890abcdef',
  roomId: undefined,
  groupId: undefined
}

const liff = {
  init: jest.fn().mockResolvedValue(undefined),
  getVersion: jest.fn().mockReturnValue('2.22.0'),
  isInClient: jest.fn().mockReturnValue(false),
  isLoggedIn: jest.fn().mockReturnValue(true),
  isExpired: jest.fn().mockReturnValue(false),
  getOS: jest.fn().mockReturnValue('web'),
  getLanguage: jest.fn().mockReturnValue('th'),
  getLineVersion: jest.fn().mockReturnValue(null),
  getContext: jest.fn().mockReturnValue(mockContext),
  getProfile: jest.fn().mockResolvedValue(mockProfile),
  getAccessToken: jest.fn().mockReturnValue('mock-access-token-xxx'),
  getDecodedIDToken: jest.fn().mockReturnValue({
    sub: mockProfile.userId,
    name: mockProfile.displayName,
    picture: mockProfile.pictureUrl
  }),
  login: jest.fn(),
  logout: jest.fn(),
  sendMessages: jest.fn().mockResolvedValue(undefined),
  shareTargetPicker: jest.fn().mockResolvedValue({ status: 'success' }),
  openWindow: jest.fn(),
  closeWindow: jest.fn(),
  scanCodeV2: jest.fn().mockResolvedValue({ value: 'QR_CODE_VALUE' }),
  getIDToken: jest.fn().mockReturnValue('mock-id-token')
}

export default liff
```

### Error Boundary สำหรับ LIFF

```typescript
// components/liff/LiffErrorBoundary.tsx
'use client'

import React, { Component, ErrorInfo, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: ReactNode
  onError?: (error: Error, errorInfo: ErrorInfo) => void
}

interface State {
  hasError: boolean
  error: Error | null
  errorInfo: ErrorInfo | null
}

export class LiffErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props)
    this.state = { hasError: false, error: null, errorInfo: null }
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    this.setState({ errorInfo })
    
    // Log error
    console.error('LIFF Error:', error, errorInfo)
    
    // Report to analytics
    this.props.onError?.(error, errorInfo)
    
    // ส่ง error ไปยัง monitoring service
    if (typeof window !== 'undefined') {
      fetch('/api/errors', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          message: error.message,
          stack: error.stack,
          componentStack: errorInfo.componentStack,
          url: window.location.href,
          timestamp: new Date().toISOString()
        })
      }).catch(() => {}) // ไม่ throw ถ้า error reporting ล้มเหลว
    }
  }

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback
      }

      return (
        <div className="min-h-screen flex items-center justify-center p-4">
          <div className="text-center max-w-sm">
            <div className="text-5xl mb-4">😢</div>
            <h2 className="text-xl font-bold text-gray-800 mb-2">
              เกิดข้อผิดพลาด
            </h2>
            <p className="text-gray-600 mb-4">
              ขออภัย เกิดข้อผิดพลาดในแอปพลิเคชัน
            </p>
            {process.env.NODE_ENV === 'development' && (
              <details className="text-left bg-red-50 p-3 rounded text-sm text-red-700 mb-4">
                <summary className="cursor-pointer font-medium">
                  รายละเอียดข้อผิดพลาด
                </summary>
                <pre className="mt-2 whitespace-pre-wrap break-all text-xs">
                  {this.state.error?.message}
                  {'\n\n'}
                  {this.state.errorInfo?.componentStack}
                </pre>
              </details>
            )}
            <button
              onClick={() => {
                this.setState({ hasError: false, error: null, errorInfo: null })
                window.location.reload()
              }}
              className="bg-green-500 text-white px-6 py-2 rounded-full font-medium"
            >
              ลองอีกครั้ง
            </button>
          </div>
        </div>
      )
    }

    return this.props.children
  }
}
```

---

## 9. Analytics in LIFF {#analytics-liff}

### LIFF Analytics Implementation

```typescript
// lib/analytics.ts
type EventCategory = 
  | 'navigation' 
  | 'interaction' 
  | 'conversion' 
  | 'error' 
  | 'performance'

interface AnalyticsEvent {
  category: EventCategory
  action: string
  label?: string
  value?: number
  userId?: string
  sessionId?: string
  timestamp: string
  metadata?: Record<string, unknown>
}

class LiffAnalytics {
  private queue: AnalyticsEvent[] = []
  private userId: string | null = null
  private sessionId: string
  private flushInterval: ReturnType<typeof setInterval> | null = null

  constructor() {
    this.sessionId = this.generateSessionId()
    this.startFlushInterval()
    this.setupPageViewTracking()
    this.setupPerformanceTracking()
  }

  private generateSessionId(): string {
    return `sess_${Date.now()}_${Math.random().toString(36).slice(2, 9)}`
  }

  setUserId(userId: string): void {
    this.userId = userId
  }

  track(
    category: EventCategory,
    action: string,
    options?: { label?: string; value?: number; metadata?: Record<string, unknown> }
  ): void {
    const event: AnalyticsEvent = {
      category,
      action,
      label: options?.label,
      value: options?.value,
      userId: this.userId ?? undefined,
      sessionId: this.sessionId,
      timestamp: new Date().toISOString(),
      metadata: options?.metadata
    }

    this.queue.push(event)
    
    // Flush immediately for important events
    if (category === 'conversion' || this.queue.length >= 10) {
      this.flush()
    }
  }

  private setupPageViewTracking(): void {
    if (typeof window === 'undefined') return

    // Track initial page view
    this.track('navigation', 'page_view', {
      label: window.location.pathname,
      metadata: {
        referrer: document.referrer,
        userAgent: navigator.userAgent.slice(0, 100)
      }
    })

    // Track navigation changes (SPA)
    const originalPushState = history.pushState
    history.pushState = (...args) => {
      originalPushState.apply(history, args)
      this.track('navigation', 'page_view', {
        label: window.location.pathname
      })
    }
  }

  private setupPerformanceTracking(): void {
    if (typeof window === 'undefined') return

    // Track Core Web Vitals
    if ('PerformanceObserver' in window) {
      // LCP (Largest Contentful Paint)
      new PerformanceObserver((entryList) => {
        const entries = entryList.getEntries()
        const lastEntry = entries[entries.length - 1]
        this.track('performance', 'lcp', {
          value: Math.round(lastEntry.startTime),
          label: 'ms'
        })
      }).observe({ type: 'largest-contentful-paint', buffered: true })

      // FID (First Input Delay)
      new PerformanceObserver((entryList) => {
        const entries = entryList.getEntries()
        entries.forEach(entry => {
          // @ts-ignore
          this.track('performance', 'fid', {
            // @ts-ignore
            value: Math.round(entry.processingStart - entry.startTime),
            label: 'ms'
          })
        })
      }).observe({ type: 'first-input', buffered: true })

      // CLS (Cumulative Layout Shift)
      let clsValue = 0
      new PerformanceObserver((entryList) => {
        for (const entry of entryList.getEntries()) {
          // @ts-ignore
          if (!entry.hadRecentInput) {
            // @ts-ignore
            clsValue += entry.value
          }
        }
      }).observe({ type: 'layout-shift', buffered: true })

      // Track CLS on page unload
      window.addEventListener('pagehide', () => {
        this.track('performance', 'cls', {
          value: Math.round(clsValue * 1000) / 1000
        })
        this.flush()
      })
    }
  }

  private async flush(): Promise<void> {
    if (this.queue.length === 0) return

    const events = [...this.queue]
    this.queue = []

    try {
      await fetch('/api/analytics', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ events }),
        keepalive: true // ส่งแม้ page ปิด
      })
    } catch (error) {
      // Put events back if flush fails
      this.queue = [...events, ...this.queue]
    }
  }

  private startFlushInterval(): void {
    this.flushInterval = setInterval(() => this.flush(), 30 * 1000)
  }

  destroy(): void {
    if (this.flushInterval) {
      clearInterval(this.flushInterval)
    }
    this.flush()
  }
}

export const analytics = new LiffAnalytics()

// React hook
export function useAnalytics() {
  return {
    track: analytics.track.bind(analytics),
    trackClick: (label: string, metadata?: Record<string, unknown>) => {
      analytics.track('interaction', 'click', { label, metadata })
    },
    trackView: (label: string) => {
      analytics.track('navigation', 'view', { label })
    },
    trackConversion: (action: string, value?: number, metadata?: Record<string, unknown>) => {
      analytics.track('conversion', action, { value, metadata })
    },
    trackError: (error: Error, context?: string) => {
      analytics.track('error', 'error_occurred', {
        label: context,
        metadata: {
          message: error.message,
          stack: error.stack?.slice(0, 500)
        }
      })
    }
  }
}
```

---

## 10. Complex LIFF App: Mini CRM {#mini-crm}

### ภาพรวม Mini CRM Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Mini CRM System                               │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   LIFF App (Next.js)                     │    │
│  │                                                           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │    │
│  │  │ Dashboard│  │Contacts  │  │ Deals    │  │Reports │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                    REST API / GraphQL                             │
│                           │                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Backend (Fastify)                       │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │    │
│  │  │  Auth    │  │  CRM     │  │Messaging │  │Analytics│  │    │
│  │  │  Service │  │  Service │  │  Service │  │ Service │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │PostgreSQL│  │  Redis   │  │ MinIO    │  │  LINE API      │  │
│  │(Primary  │  │(Cache)   │  │(Files)   │  │  (Messaging)   │  │
│  │ Data)    │  │          │  │          │  │                │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Database Schema (Prisma)

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "fullTextIndex"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id          String   @id @default(cuid())
  lineUserId  String   @unique @map("line_user_id")
  displayName String   @map("display_name")
  pictureUrl  String?  @map("picture_url")
  email       String?  @unique
  phone       String?
  role        UserRole @default(AGENT)
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  contacts    Contact[]
  deals       Deal[]
  activities  Activity[]
  notes       Note[]

  @@map("users")
}

enum UserRole {
  ADMIN
  MANAGER
  AGENT
}

model Contact {
  id          String        @id @default(cuid())
  lineUserId  String?       @map("line_user_id")
  displayName String        @map("display_name")
  email       String?
  phone       String?
  company     String?
  position    String?
  status      ContactStatus @default(PROSPECT)
  source      String?
  tags        String[]
  customFields Json?        @map("custom_fields")
  notes       String?
  createdById String        @map("created_by_id")
  assignedToId String?      @map("assigned_to_id")
  createdAt   DateTime      @default(now()) @map("created_at")
  updatedAt   DateTime      @updatedAt @map("updated_at")
  lastContactAt DateTime?   @map("last_contact_at")

  createdBy   User          @relation(fields: [createdById], references: [id])
  deals       Deal[]
  activities  Activity[]
  notes_      Note[]
  tags_       ContactTag[]

  @@index([status])
  @@index([createdById])
  @@index([assignedToId])
  @@map("contacts")
}

enum ContactStatus {
  PROSPECT
  ACTIVE
  INACTIVE
  CHURNED
}

model Deal {
  id          String     @id @default(cuid())
  title       String
  contactId   String     @map("contact_id")
  ownerId     String     @map("owner_id")
  stage       DealStage  @default(LEAD)
  value       Float      @default(0)
  currency    String     @default("THB")
  probability Int        @default(10)
  expectedClose DateTime? @map("expected_close")
  wonAt       DateTime?  @map("won_at")
  lostAt      DateTime?  @map("lost_at")
  lostReason  String?    @map("lost_reason")
  createdAt   DateTime   @default(now()) @map("created_at")
  updatedAt   DateTime   @updatedAt @map("updated_at")

  contact     Contact    @relation(fields: [contactId], references: [id])
  owner       User       @relation(fields: [ownerId], references: [id])
  activities  Activity[]

  @@index([stage])
  @@index([ownerId])
  @@map("deals")
}

enum DealStage {
  LEAD
  QUALIFIED
  PROPOSAL
  NEGOTIATION
  WON
  LOST
}

model Activity {
  id          String       @id @default(cuid())
  type        ActivityType
  subject     String
  body        String?
  contactId   String?      @map("contact_id")
  dealId      String?      @map("deal_id")
  userId      String       @map("user_id")
  scheduledAt DateTime?    @map("scheduled_at")
  completedAt DateTime?    @map("completed_at")
  createdAt   DateTime     @default(now()) @map("created_at")

  contact     Contact?     @relation(fields: [contactId], references: [id])
  deal        Deal?        @relation(fields: [dealId], references: [id])
  user        User         @relation(fields: [userId], references: [id])

  @@index([contactId])
  @@index([dealId])
  @@map("activities")
}

enum ActivityType {
  CALL
  EMAIL
  MEETING
  LINE_MESSAGE
  NOTE
  TASK
}

model Note {
  id        String   @id @default(cuid())
  content   String
  contactId String?  @map("contact_id")
  userId    String   @map("user_id")
  createdAt DateTime @default(now()) @map("created_at")

  contact   Contact? @relation(fields: [contactId], references: [id])
  user      User     @relation(fields: [userId], references: [id])

  @@map("notes")
}

model ContactTag {
  contactId String  @map("contact_id")
  tag       String
  contact   Contact @relation(fields: [contactId], references: [id])

  @@id([contactId, tag])
  @@map("contact_tags")
}
```

### Contacts API

```typescript
// app/api/crm/contacts/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { z } from 'zod'

const CreateContactSchema = z.object({
  displayName: z.string().min(1).max(100),
  email: z.string().email().optional(),
  phone: z.string().optional(),
  company: z.string().optional(),
  position: z.string().optional(),
  status: z.enum(['PROSPECT', 'ACTIVE', 'INACTIVE', 'CHURNED']).default('PROSPECT'),
  source: z.string().optional(),
  tags: z.array(z.string()).default([]),
  notes: z.string().optional()
})

export async function GET(request: NextRequest) {
  const userId = request.headers.get('x-line-user-id')
  if (!userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const { searchParams } = new URL(request.url)
  const page = parseInt(searchParams.get('page') ?? '1')
  const limit = parseInt(searchParams.get('limit') ?? '20')
  const search = searchParams.get('search')
  const status = searchParams.get('status')
  const tag = searchParams.get('tag')

  const user = await prisma.user.findUnique({ where: { lineUserId: userId } })
  if (!user) return NextResponse.json({ error: 'User not found' }, { status: 404 })

  const where: Record<string, unknown> = {
    createdById: user.id
  }

  if (search) {
    where.OR = [
      { displayName: { contains: search, mode: 'insensitive' } },
      { email: { contains: search, mode: 'insensitive' } },
      { phone: { contains: search } },
      { company: { contains: search, mode: 'insensitive' } }
    ]
  }

  if (status) where.status = status
  if (tag) where.tags = { has: tag }

  const [contacts, total] = await Promise.all([
    prisma.contact.findMany({
      where,
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { updatedAt: 'desc' },
      include: {
        _count: {
          select: { deals: true, activities: true }
        }
      }
    }),
    prisma.contact.count({ where })
  ])

  return NextResponse.json({
    data: contacts,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit)
    }
  })
}

export async function POST(request: NextRequest) {
  const userId = request.headers.get('x-line-user-id')
  if (!userId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })

  const body = await request.json()
  const validation = CreateContactSchema.safeParse(body)

  if (!validation.success) {
    return NextResponse.json(
      { error: 'Validation failed', details: validation.error.flatten() },
      { status: 400 }
    )
  }

  const user = await prisma.user.findUnique({ where: { lineUserId: userId } })
  if (!user) return NextResponse.json({ error: 'User not found' }, { status: 404 })

  const contact = await prisma.contact.create({
    data: {
      ...validation.data,
      createdById: user.id,
      assignedToId: user.id
    }
  })

  return NextResponse.json(contact, { status: 201 })
}
```

---

## 11. Complete Next.js + LIFF Project {#complete-project}

### Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  # Next.js LIFF App
  web:
    build:
      context: ./apps/web
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://crm:crm_pass@postgres:5432/crm_db
      - REDIS_URL=redis://redis:6379
      - NEXT_PUBLIC_LIFF_ID=${LIFF_ID}
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - NEXTAUTH_URL=https://your-domain.com
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # PostgreSQL
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: crm_db
      POSTGRES_USER: crm
      POSTGRES_PASSWORD: crm_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U crm -d crm_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass redis_pass
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis_pass", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Nginx reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/nginx/ssl:/etc/nginx/ssl
      - certbot_data:/var/www/certbot
    depends_on:
      - web
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  certbot_data:
```

### Dockerfile สำหรับ Next.js

```dockerfile
# apps/web/Dockerfile
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Dependencies
FROM base AS deps
COPY package*.json ./
COPY prisma ./prisma/
RUN npm ci --only=production

# Build
FROM base AS builder
COPY package*.json ./
RUN npm ci
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN npx prisma generate
RUN npm run build

# Runner
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder /app/prisma ./prisma
COPY --from=deps /app/node_modules/.prisma ./node_modules/.prisma

USER nextjs

EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy LINE Mini CRM

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run Prisma migrations
        env:
          DATABASE_URL: postgresql://test:test_pass@localhost:5432/test_db
        run: npx prisma migrate deploy
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test_pass@localhost:5432/test_db
          NEXT_PUBLIC_LIFF_ID: ${{ secrets.TEST_LIFF_ID }}
        run: npm test -- --coverage
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,format=short
            type=ref,event=branch
            latest
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./apps/web
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/mini-crm
            docker compose pull
            docker compose up -d --no-deps web
            docker compose run --rm web npx prisma migrate deploy
            docker system prune -f
```

### Nginx Configuration

```nginx
# docker/nginx/nginx.conf
worker_processes auto;
error_log /var/log/nginx/error.log warn;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/x-javascript application/xml 
               application/javascript application/json;

    # Security
    server_tokens off;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy strict-origin-when-cross-origin;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=60r/m;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=10r/m;

    upstream nextjs {
        server web:3000;
        keepalive 32;
    }

    # HTTP -> HTTPS redirect
    server {
        listen 80;
        server_name your-domain.com;
        
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
        
        location / {
            return 301 https://$host$request_uri;
        }
    }

    # HTTPS
    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;

        # HSTS
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # API rate limiting
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://nextjs;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }

        # Auth rate limiting
        location /api/auth/ {
            limit_req zone=auth burst=5 nodelay;
            proxy_pass http://nextjs;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Static files - cache aggressively
        location /_next/static/ {
            proxy_pass http://nextjs;
            proxy_http_version 1.1;
            add_header Cache-Control "public, max-age=31536000, immutable";
        }

        # Next.js App
        location / {
            proxy_pass http://nextjs;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }
    }
}
```

### Monitoring และ Health Checks

```typescript
// app/api/health/route.ts
import { NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'
import { redis } from '@/lib/redis'

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy'
  timestamp: string
  version: string
  checks: {
    database: CheckResult
    redis: CheckResult
    lineApi: CheckResult
  }
  uptime: number
}

interface CheckResult {
  status: 'ok' | 'error'
  latency?: number
  error?: string
}

export async function GET(): Promise<NextResponse<HealthStatus>> {
  const startTime = Date.now()
  
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkLineApi()
  ])

  const [dbCheck, redisCheck, lineCheck] = checks.map(result => {
    if (result.status === 'fulfilled') return result.value
    return { status: 'error' as const, error: String(result.reason) }
  })

  const allHealthy = [dbCheck, redisCheck, lineCheck].every(c => c.status === 'ok')
  const anyUnhealthy = [dbCheck, redisCheck, lineCheck].some(c => c.status === 'error')

  const overallStatus = allHealthy ? 'healthy' : 
                        anyUnhealthy ? 'unhealthy' : 'degraded'

  const response: HealthStatus = {
    status: overallStatus,
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version ?? '1.0.0',
    checks: {
      database: dbCheck,
      redis: redisCheck,
      lineApi: lineCheck
    },
    uptime: process.uptime()
  }

  return NextResponse.json(response, {
    status: overallStatus === 'unhealthy' ? 503 : 200
  })
}

async function checkDatabase(): Promise<CheckResult> {
  const start = Date.now()
  try {
    await prisma.$queryRaw`SELECT 1`
    return { status: 'ok', latency: Date.now() - start }
  } catch (error) {
    return { status: 'error', error: String(error) }
  }
}

async function checkRedis(): Promise<CheckResult> {
  const start = Date.now()
  try {
    await redis.ping()
    return { status: 'ok', latency: Date.now() - start }
  } catch (error) {
    return { status: 'error', error: String(error) }
  }
}

async function checkLineApi(): Promise<CheckResult> {
  const start = Date.now()
  try {
    const response = await fetch('https://api.line.me/v2/bot/info', {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      },
      signal: AbortSignal.timeout(5000)
    })
    
    if (!response.ok) {
      return { status: 'error', error: `HTTP ${response.status}` }
    }
    
    return { status: 'ok', latency: Date.now() - start }
  } catch (error) {
    return { status: 'error', error: String(error) }
  }
}
```

### Performance Monitoring ด้วย OpenTelemetry

```typescript
// instrumentation.ts
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { Resource } from '@opentelemetry/resources'
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions'
import { PrismaInstrumentation } from '@prisma/instrumentation'
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http'

export function register() {
  const sdk = new NodeSDK({
    resource: new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]: 'line-mini-crm',
      [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version,
      [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV
    }),
    traceExporter: new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces'
    }),
    instrumentations: [
      new HttpInstrumentation(),
      new PrismaInstrumentation()
    ]
  })

  sdk.start()

  process.on('SIGTERM', () => {
    sdk.shutdown().then(() => process.exit(0))
  })
}
```

### สรุป: Advanced LIFF Best Practices

```
✅ Architecture Best Practices:
   - แยก Server Components และ Client Components ชัดเจน
   - ใช้ Dynamic Imports สำหรับ heavy components
   - Implement proper error boundaries
   - ใช้ service workers สำหรับ offline support

✅ Performance:
   - Bundle size < 200KB initial JS
   - LCP < 2.5s
   - FID < 100ms
   - CLS < 0.1

✅ Security:
   - Always verify LINE access tokens บน server
   - ใช้ HTTPS everywhere
   - Implement rate limiting
   - Never trust client-side data

✅ Developer Experience:
   - Mock LIFF SDK ใน development
   - Debug panel สำหรับ troubleshooting
   - Comprehensive error logging
   - TypeScript strict mode
```

---

*Part 86 จบแล้ว - ต่อไปใน Part 87: LINE Mini App*
