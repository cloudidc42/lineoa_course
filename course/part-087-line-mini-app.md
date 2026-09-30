# Part 87: LINE Mini App

## สารบัญ

1. [LINE Mini App Overview](#overview)
2. [ความแตกต่าง LIFF vs Mini App](#liff-vs-mini-app)
3. [Use Cases ของ Mini App](#use-cases)
4. [การตั้งค่า Mini App](#setup)
5. [Mini App SDK](#sdk)
6. [Mini App Components และ UI Kit](#ui-kit)
7. [Navigation ใน Mini App](#navigation)
8. [Payment ใน Mini App](#payment)
9. [การเผยแพร่ Mini App](#distribution)
10. [Complete Mini App: Online Store](#online-store)

---

## 1. LINE Mini App Overview {#overview}

### LINE Mini App คืออะไร

LINE Mini App เป็น Web Application ที่ทำงานภายใน LINE Application โดยตรง มีลักษณะคล้าย Native App แต่พัฒนาด้วย Web Technologies โดยมี UI/UX ที่ออกแบบมาให้เข้ากับ LINE Platform

```
┌─────────────────────────────────────────────────────────┐
│                    LINE App                              │
│                                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │              LINE Mini App                         │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  Header Bar (LINE Native)                    │  │  │
│  │  │  ← Back  [App Name]  Share  ⋮               │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │                                                     │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  App Content (WebView)                       │  │  │
│  │  │                                              │  │  │
│  │  │  [Your Web Application Here]                 │  │  │
│  │  │                                              │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │                                                     │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  Bottom Tab Bar (LINE Native)                │  │  │
│  │  │  🏠 Home  🔍 Search  🛒 Cart  👤 Profile    │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### คุณสมบัติหลักของ LINE Mini App

| คุณสมบัติ | รายละเอียด |
|-----------|------------|
| Native UI Components | ใช้ UI Kit ของ LINE ได้โดยตรง |
| LINE Pay Integration | ชำระเงินผ่าน LINE Pay ได้ |
| LINE Login | Login ด้วย LINE account |
| Push Notifications | ส่ง push notifications |
| QR Code Scanner | สแกน QR code ในตัว |
| Location Services | เข้าถึง GPS location |
| Camera Access | ถ่ายรูปและอัปโหลด |
| Offline Support | ทำงานได้แบบ offline บางส่วน |
| App Sharing | แชร์ app ผ่าน LINE Message |
| Analytics | built-in analytics dashboard |

---

## 2. ความแตกต่าง LIFF vs Mini App {#liff-vs-mini-app}

### เปรียบเทียบโดยละเอียด

```
┌──────────────────────────────────────────────────────────────────┐
│              LIFF vs LINE Mini App                                │
├────────────────────────┬────────────────────────────────────────┤
│ Feature                │ LIFF            │ LINE Mini App         │
├────────────────────────┼─────────────────┼───────────────────────┤
│ เปิดได้จาก            │ OA Message,     │ LINE App Home,        │
│                        │ Rich Menu, URL  │ Search, OA Message    │
├────────────────────────┼─────────────────┼───────────────────────┤
│ UI Framework           │ ไม่มี (เสรี)   │ LINE UI Kit (บังคับ) │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Bottom Navigation      │ Custom          │ Native LINE UI        │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Header Bar             │ ไม่มี           │ Native LINE UI        │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Discovery              │ จาก OA เท่านั้น │ LINE App Store        │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Offline Support        │ Manual (SW)     │ Built-in              │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Payment                │ External/LINEPay│ LINE Pay (native)     │
├────────────────────────┼─────────────────┼───────────────────────┤
│ App Store Listing      │ ไม่ได้          │ ได้                   │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Analytics              │ Custom          │ Built-in LINE         │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Approval Process       │ เร็ว            │ Review Required       │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Target Users           │ OA followers    │ All LINE users        │
├────────────────────────┼─────────────────┼───────────────────────┤
│ Availability           │ All countries   │ Thailand/Japan/Taiwan │
│                        │                 │ (expanding)           │
└──────────────────────────────────────────────────────────────────┘
```

### เมื่อไหรควรใช้อะไร

```
ใช้ LIFF เมื่อ:
✅ ต้องการ custom UI อย่างอิสระ
✅ ต้องการ deploy เร็ว ไม่ต้องรอ review
✅ ใช้สำหรับ OA followers เท่านั้น
✅ ต้องการ integrate กับ chatbot
✅ App สำหรับ internal use

ใช้ LINE Mini App เมื่อ:
✅ ต้องการ reach ผู้ใช้ใหม่ผ่าน LINE App Store
✅ ต้องการ native LINE payment experience
✅ ต้องการ bottom navigation แบบ native
✅ สร้าง marketplace/e-commerce
✅ Public-facing app สำหรับ general users
```

---

## 3. Use Cases ของ Mini App {#use-cases}

### Use Cases ยอดนิยม

```
1. E-Commerce
   - ร้านค้าออนไลน์
   - Marketplace
   - Flash sale platform
   - Product catalog

2. Food & Delivery
   - Food ordering
   - Restaurant reservation
   - Grocery delivery
   - Meal kit subscription

3. Financial Services
   - Insurance purchase
   - Loan application
   - Investment platform
   - Bill payment

4. Healthcare
   - Doctor appointment booking
   - Telemedicine
   - Medicine delivery
   - Health tracking

5. Travel & Tourism
   - Hotel booking
   - Tour package
   - Transportation
   - Attraction tickets

6. Loyalty & Rewards
   - Points collection
   - Member card
   - Coupon/voucher
   - Privilege program
```

---

## 4. การตั้งค่า Mini App {#setup}

### ขั้นตอนการสมัคร

```
1. เข้า LINE Developers Console
   https://developers.line.biz/

2. สร้าง Provider (ถ้ายังไม่มี)
   - กด "Create a Provider"
   - กรอกชื่อ Provider

3. สร้าง LINE Mini App Channel
   - เลือก Provider
   - กด "Create a new channel"
   - เลือก "LINE Mini App"
   - กรอกข้อมูล:
     * Channel name: ชื่อ Mini App
     * Channel description: คำอธิบาย
     * Category: ประเภท
     * Region: Thailand
     * Email: อีเมลผู้ติดต่อ

4. กรอกข้อมูล Mini App
   - App icon: รูป icon 512x512
   - App name: ชื่อที่แสดงใน LINE
   - Short description: คำอธิบายสั้น
   - Long description: คำอธิบายยาว
   - Screenshots: รูป screenshot 2-8 ใบ
   - Category: ประเภท app

5. ตั้งค่า Endpoint URL
   - Endpoint URL: https://your-domain.com/mini-app
   - ต้องเป็น HTTPS

6. ส่งเพื่อ Review
   - กด "Submit for review"
   - รอ LINE team ตรวจสอบ 3-5 วันทำการ
```

### Project Structure

```
line-mini-app/
├── public/
│   ├── index.html
│   ├── manifest.json
│   └── assets/
├── src/
│   ├── pages/
│   │   ├── Home.tsx
│   │   ├── Products.tsx
│   │   ├── ProductDetail.tsx
│   │   ├── Cart.tsx
│   │   ├── Checkout.tsx
│   │   ├── Orders.tsx
│   │   └── Profile.tsx
│   ├── components/
│   │   ├── layout/
│   │   │   ├── AppHeader.tsx
│   │   │   ├── BottomNavigation.tsx
│   │   │   └── PageContainer.tsx
│   │   ├── product/
│   │   │   ├── ProductCard.tsx
│   │   │   ├── ProductGrid.tsx
│   │   │   └── ProductSearch.tsx
│   │   └── common/
│   │       ├── LoadingSpinner.tsx
│   │       ├── ErrorMessage.tsx
│   │       └── EmptyState.tsx
│   ├── hooks/
│   │   ├── useMiniApp.ts
│   │   ├── useCart.ts
│   │   └── usePayment.ts
│   ├── store/
│   │   ├── cartStore.ts
│   │   └── userStore.ts
│   ├── services/
│   │   ├── api.ts
│   │   ├── payment.ts
│   │   └── auth.ts
│   └── App.tsx
├── package.json
└── vite.config.ts
```

---

## 5. Mini App SDK {#sdk}

### การติดตั้ง SDK

```bash
# ติดตั้ง LINE Mini App SDK
npm install @line/mini-app-sdk

# หรือสำหรับ React
npm install @line/mini-app-sdk react react-dom

# TypeScript types
npm install --save-dev @types/line__mini-app-sdk
```

### SDK Initialization

```typescript
// src/lib/miniapp.ts
import MiniApp from '@line/mini-app-sdk'

interface MiniAppConfig {
  channelId: string
  version?: string
}

class MiniAppClient {
  private miniApp: typeof MiniApp | null = null
  private initialized = false

  async init(config: MiniAppConfig): Promise<void> {
    if (this.initialized) return

    try {
      await MiniApp.ready()
      
      // Set up event listeners
      MiniApp.onShow(() => {
        console.log('Mini App shown')
        this.onAppShow()
      })

      MiniApp.onHide(() => {
        console.log('Mini App hidden')
        this.onAppHide()
      })

      MiniApp.onError((error) => {
        console.error('Mini App error:', error)
        this.onAppError(error)
      })

      this.miniApp = MiniApp
      this.initialized = true
      
      console.log('Mini App initialized', {
        version: MiniApp.version,
        platform: MiniApp.platform
      })
    } catch (error) {
      throw new Error(`Failed to initialize Mini App: ${error}`)
    }
  }

  private onAppShow(): void {
    // รีเฟรช data เมื่อ app กลับมา foreground
    window.dispatchEvent(new Event('miniapp:show'))
  }

  private onAppHide(): void {
    // บันทึก state เมื่อ app ไป background
    window.dispatchEvent(new Event('miniapp:hide'))
  }

  private onAppError(error: unknown): void {
    // Log error ไปยัง monitoring
    console.error('Mini App critical error:', error)
  }

  getMiniApp(): typeof MiniApp {
    if (!this.miniApp) throw new Error('Mini App not initialized')
    return this.miniApp
  }

  isInitialized(): boolean {
    return this.initialized
  }
}

export const miniAppClient = new MiniAppClient()
export { MiniApp }
```

### User Authentication

```typescript
// src/services/auth.ts
import MiniApp from '@line/mini-app-sdk'

interface UserProfile {
  userId: string
  displayName: string
  pictureUrl?: string
  language: string
}

interface AuthResult {
  profile: UserProfile
  accessToken: string
  idToken: string
}

export async function authenticateUser(): Promise<AuthResult> {
  try {
    // ขอ permission จาก user
    const permissionResult = await MiniApp.requestPermission({
      permission: 'user_profile'
    })

    if (!permissionResult.granted) {
      throw new Error('User denied profile permission')
    }

    // รับ user profile
    const profile = await MiniApp.getUserProfile()
    
    // รับ access token
    const accessToken = await MiniApp.getAccessToken()
    const idToken = await MiniApp.getIDToken()

    return {
      profile: {
        userId: profile.userId,
        displayName: profile.displayName,
        pictureUrl: profile.pictureUrl,
        language: MiniApp.language ?? 'th'
      },
      accessToken,
      idToken
    }
  } catch (error) {
    throw new Error(`Authentication failed: ${error}`)
  }
}

// Verify token บน server side
export async function verifyServerToken(idToken: string): Promise<UserProfile> {
  const response = await fetch('/api/auth/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ idToken })
  })

  if (!response.ok) {
    throw new Error('Token verification failed')
  }

  return response.json()
}
```

### Camera และ QR Scanner

```typescript
// src/services/camera.ts
import MiniApp from '@line/mini-app-sdk'

interface ScanResult {
  value: string
  type: 'qr' | 'barcode'
}

interface PhotoResult {
  url: string
  width: number
  height: number
  size: number
}

// สแกน QR Code
export async function scanQRCode(): Promise<ScanResult> {
  try {
    const result = await MiniApp.scanCode()
    
    return {
      value: result.value,
      type: 'qr'
    }
  } catch (error) {
    if (error instanceof Error && error.message.includes('canceled')) {
      throw new Error('USER_CANCELED')
    }
    throw error
  }
}

// ถ่ายรูป
export async function takePhoto(options?: {
  maxWidth?: number
  maxHeight?: number
  quality?: number
}): Promise<PhotoResult> {
  try {
    const result = await MiniApp.requestCamera({
      action: 'take_photo',
      maxWidth: options?.maxWidth ?? 1080,
      maxHeight: options?.maxHeight ?? 1920,
      quality: options?.quality ?? 80
    })

    return {
      url: result.url,
      width: result.width,
      height: result.height,
      size: result.size
    }
  } catch (error) {
    if (error instanceof Error && error.message.includes('denied')) {
      throw new Error('CAMERA_PERMISSION_DENIED')
    }
    throw error
  }
}

// เลือกรูปจาก gallery
export async function pickFromGallery(options?: {
  multiple?: boolean
  maxCount?: number
}): Promise<PhotoResult[]> {
  try {
    const result = await MiniApp.requestGallery({
      multiple: options?.multiple ?? false,
      maxCount: options?.maxCount ?? 1
    })

    return result.photos.map(photo => ({
      url: photo.url,
      width: photo.width,
      height: photo.height,
      size: photo.size
    }))
  } catch (error) {
    throw error
  }
}
```

### Location Services

```typescript
// src/services/location.ts
import MiniApp from '@line/mini-app-sdk'

interface Location {
  latitude: number
  longitude: number
  accuracy: number
  altitude?: number
  address?: string
}

export async function getCurrentLocation(options?: {
  highAccuracy?: boolean
  timeout?: number
}): Promise<Location> {
  // ขอ permission
  const permission = await MiniApp.requestPermission({
    permission: 'location'
  })

  if (!permission.granted) {
    throw new Error('LOCATION_PERMISSION_DENIED')
  }

  const position = await MiniApp.getLocation({
    enableHighAccuracy: options?.highAccuracy ?? false,
    timeout: options?.timeout ?? 10000
  })

  // รับ address จาก reverse geocoding (optional)
  let address: string | undefined
  try {
    address = await reverseGeocode(position.coords.latitude, position.coords.longitude)
  } catch {
    // ไม่ error ถ้า geocoding ล้มเหลว
  }

  return {
    latitude: position.coords.latitude,
    longitude: position.coords.longitude,
    accuracy: position.coords.accuracy,
    altitude: position.coords.altitude ?? undefined,
    address
  }
}

async function reverseGeocode(lat: number, lng: number): Promise<string> {
  const response = await fetch(
    `https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lng}&format=json`
  )
  const data = await response.json()
  return data.display_name
}
```

---

## 6. Mini App Components และ UI Kit {#ui-kit}

### LINE UI Kit Components

```typescript
// ติดตั้ง LINE UI Kit
// npm install @line/line-ui

// src/components/LineUIComponents.tsx
import {
  Button,
  Card,
  List,
  ListItem,
  Avatar,
  Badge,
  Chip,
  Divider,
  Icon,
  Image,
  Input,
  Modal,
  Snackbar,
  Switch,
  Tabs,
  TabItem,
  SearchBar,
  ActionSheet,
  ActionSheetItem,
  BottomSheet
} from '@line/line-ui'

// Product Card ด้วย LINE UI Kit
export function ProductCard({ product }: { product: Product }) {
  return (
    <Card
      header={
        <Image
          src={product.imageUrl}
          alt={product.name}
          aspectRatio="4:3"
          objectFit="cover"
        />
      }
      footer={
        <div className="flex justify-between items-center">
          <span className="text-lg font-bold text-green-600">
            ฿{product.price.toLocaleString()}
          </span>
          <Button
            variant="primary"
            size="small"
            onClick={() => addToCart(product)}
          >
            เพิ่มลงตะกร้า
          </Button>
        </div>
      }
    >
      <div className="p-3">
        <h3 className="font-semibold text-gray-800 line-clamp-2">
          {product.name}
        </h3>
        <p className="text-sm text-gray-500 mt-1 line-clamp-1">
          {product.description}
        </p>
        <div className="flex items-center gap-2 mt-2">
          <Badge variant="success" size="small">
            {product.category}
          </Badge>
          {product.isNew && (
            <Badge variant="warning" size="small">NEW</Badge>
          )}
        </div>
      </div>
    </Card>
  )
}
```

### Custom Components

```typescript
// src/components/product/ProductGrid.tsx
import { useState, useCallback } from 'react'
import { SearchBar } from '@line/line-ui'
import { ProductCard } from './ProductCard'
import { useProducts } from '@/hooks/useProducts'
import { LoadingGrid } from '@/components/common/LoadingGrid'
import { EmptyState } from '@/components/common/EmptyState'

interface ProductGridProps {
  categoryId?: string
  onProductSelect?: (product: Product) => void
}

export function ProductGrid({ categoryId, onProductSelect }: ProductGridProps) {
  const [search, setSearch] = useState('')
  const [page, setPage] = useState(1)

  const { data, isLoading, isError, fetchNextPage, hasNextPage } = useProducts({
    categoryId,
    search,
    page
  })

  const handleSearch = useCallback((value: string) => {
    setSearch(value)
    setPage(1)
  }, [])

  if (isLoading && page === 1) return <LoadingGrid />
  if (isError) return <ErrorMessage message="ไม่สามารถโหลดสินค้าได้" />
  if (!data?.items.length) return <EmptyState message="ไม่พบสินค้า" />

  return (
    <div>
      <div className="px-4 py-2 sticky top-0 bg-white z-10 shadow-sm">
        <SearchBar
          placeholder="ค้นหาสินค้า..."
          value={search}
          onChange={handleSearch}
          clearable
        />
      </div>

      <div className="grid grid-cols-2 gap-3 p-4">
        {data.items.map(product => (
          <ProductCard
            key={product.id}
            product={product}
            onClick={() => onProductSelect?.(product)}
          />
        ))}
      </div>

      {hasNextPage && (
        <div className="p-4 text-center">
          <button
            onClick={() => fetchNextPage()}
            className="text-green-600 font-medium"
          >
            โหลดเพิ่มเติม
          </button>
        </div>
      )}
    </div>
  )
}
```

---

## 7. Navigation ใน Mini App {#navigation}

### Bottom Navigation Setup

```typescript
// src/components/layout/BottomNavigation.tsx
import { BottomNavigation, BottomNavigationItem } from '@line/line-ui'
import { useNavigate, useLocation } from 'react-router-dom'
import { 
  HomeIcon, 
  GridIcon, 
  ShoppingCartIcon, 
  PackageIcon, 
  UserIcon 
} from '@/components/icons'
import { useCartStore } from '@/store/cartStore'

export function AppBottomNavigation() {
  const navigate = useNavigate()
  const location = useLocation()
  const cartCount = useCartStore(s => s.items.length)

  const tabs = [
    { path: '/', icon: <HomeIcon />, label: 'หน้าแรก' },
    { path: '/products', icon: <GridIcon />, label: 'สินค้า' },
    {
      path: '/cart',
      icon: <ShoppingCartIcon />,
      label: 'ตะกร้า',
      badge: cartCount > 0 ? cartCount : undefined
    },
    { path: '/orders', icon: <PackageIcon />, label: 'คำสั่งซื้อ' },
    { path: '/profile', icon: <UserIcon />, label: 'โปรไฟล์' }
  ]

  return (
    <BottomNavigation>
      {tabs.map(tab => (
        <BottomNavigationItem
          key={tab.path}
          icon={tab.icon}
          label={tab.label}
          badge={tab.badge}
          active={location.pathname === tab.path}
          onClick={() => navigate(tab.path)}
        />
      ))}
    </BottomNavigation>
  )
}
```

### Page Container

```typescript
// src/components/layout/PageContainer.tsx
import { Header } from '@line/line-ui'
import { useNavigate } from 'react-router-dom'
import { AppBottomNavigation } from './BottomNavigation'

interface PageContainerProps {
  title?: string
  showBack?: boolean
  showShare?: boolean
  onShare?: () => void
  children: React.ReactNode
  hideNavigation?: boolean
  headerRight?: React.ReactNode
  onScroll?: (scrollTop: number) => void
}

export function PageContainer({
  title,
  showBack = false,
  showShare = false,
  onShare,
  children,
  hideNavigation = false,
  headerRight
}: PageContainerProps) {
  const navigate = useNavigate()

  return (
    <div className="flex flex-col min-h-screen bg-gray-50">
      {title && (
        <Header
          title={title}
          left={showBack ? (
            <button onClick={() => navigate(-1)} className="p-2">
              ←
            </button>
          ) : undefined}
          right={headerRight ?? (showShare ? (
            <button onClick={onShare} className="p-2">
              Share
            </button>
          ) : undefined)}
        />
      )}

      <main className="flex-1 overflow-auto pb-safe">
        {children}
      </main>

      {!hideNavigation && <AppBottomNavigation />}
    </div>
  )
}
```

### Router Configuration

```typescript
// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import { lazy, Suspense } from 'react'
import { PageContainer } from '@/components/layout/PageContainer'
import { LoadingSpinner } from '@/components/common/LoadingSpinner'

const Home = lazy(() => import('@/pages/Home'))
const Products = lazy(() => import('@/pages/Products'))
const ProductDetail = lazy(() => import('@/pages/ProductDetail'))
const Cart = lazy(() => import('@/pages/Cart'))
const Checkout = lazy(() => import('@/pages/Checkout'))
const Orders = lazy(() => import('@/pages/Orders'))
const OrderDetail = lazy(() => import('@/pages/OrderDetail'))
const Profile = lazy(() => import('@/pages/Profile'))

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner fullscreen />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/products" element={<Products />} />
          <Route path="/products/:id" element={<ProductDetail />} />
          <Route path="/cart" element={<Cart />} />
          <Route path="/checkout" element={<Checkout />} />
          <Route path="/orders" element={<Orders />} />
          <Route path="/orders/:id" element={<OrderDetail />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}

export default App
```

---

## 8. Payment ใน Mini App {#payment}

### LINE Pay Integration

```typescript
// src/services/payment.ts
import MiniApp from '@line/mini-app-sdk'

interface PaymentRequest {
  orderId: string
  amount: number
  currency: string
  orderName: string
  items: PaymentItem[]
  redirectUrls?: {
    confirmUrl: string
    cancelUrl: string
  }
}

interface PaymentItem {
  id: string
  name: string
  imageUrl?: string
  quantity: number
  price: number
}

interface PaymentResult {
  transactionId: string
  status: 'success' | 'failed' | 'canceled'
  orderId: string
}

export async function requestPayment(payment: PaymentRequest): Promise<PaymentResult> {
  try {
    // สร้าง payment request บน server
    const serverResponse = await fetch('/api/payments/request', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${await MiniApp.getAccessToken()}`
      },
      body: JSON.stringify(payment)
    })

    if (!serverResponse.ok) {
      throw new Error('Failed to create payment request')
    }

    const { paymentUrl, transactionId } = await serverResponse.json()

    // เปิด LINE Pay
    const result = await MiniApp.openLinePay({
      paymentUrl,
      redirectUrl: payment.redirectUrls?.confirmUrl ?? `${window.location.origin}/payment/success`
    })

    if (result.status === 'SUCCESS') {
      // Confirm payment บน server
      await confirmPayment(transactionId)

      return {
        transactionId,
        status: 'success',
        orderId: payment.orderId
      }
    } else if (result.status === 'CANCEL') {
      return {
        transactionId,
        status: 'canceled',
        orderId: payment.orderId
      }
    } else {
      return {
        transactionId,
        status: 'failed',
        orderId: payment.orderId
      }
    }
  } catch (error) {
    throw new Error(`Payment failed: ${error}`)
  }
}

async function confirmPayment(transactionId: string): Promise<void> {
  const response = await fetch(`/api/payments/confirm/${transactionId}`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${await MiniApp.getAccessToken()}`
    }
  })

  if (!response.ok) {
    throw new Error('Payment confirmation failed')
  }
}
```

### Server-side Payment Handler

```typescript
// api/payments/request.ts (Backend)
import { Client } from '@line/pay'
import { prisma } from '@/lib/prisma'

const linePay = new Client({
  channelId: process.env.LINE_PAY_CHANNEL_ID!,
  channelSecretKey: process.env.LINE_PAY_CHANNEL_SECRET!,
  isSandbox: process.env.NODE_ENV !== 'production'
})

export async function createPaymentRequest(req: Request, res: Response) {
  const userId = req.headers['x-line-user-id'] as string
  const body = req.body as PaymentRequest

  // สร้าง order ในฐานข้อมูล
  const order = await prisma.order.create({
    data: {
      userId,
      status: 'PENDING',
      totalAmount: body.amount,
      currency: body.currency,
      items: {
        createMany: {
          data: body.items.map(item => ({
            productId: item.id,
            quantity: item.quantity,
            price: item.price,
            name: item.name
          }))
        }
      }
    }
  })

  // Request ไปยัง LINE Pay
  const linePayResponse = await linePay.request({
    amount: body.amount,
    currency: body.currency as 'THB',
    orderId: order.id,
    packages: [{
      id: `pkg_${order.id}`,
      amount: body.amount,
      name: body.orderName,
      products: body.items.map(item => ({
        id: item.id,
        name: item.name,
        imageUrl: item.imageUrl,
        quantity: item.quantity,
        price: item.price
      }))
    }],
    redirectUrls: {
      confirmUrl: `${process.env.APP_URL}/api/payments/callback/confirm`,
      cancelUrl: `${process.env.APP_URL}/api/payments/callback/cancel`
    }
  })

  if (linePayResponse.returnCode !== '0000') {
    throw new Error(`LINE Pay error: ${linePayResponse.returnMessage}`)
  }

  // บันทึก transaction
  await prisma.payment.create({
    data: {
      orderId: order.id,
      transactionId: linePayResponse.info.transactionId.toString(),
      paymentUrl: linePayResponse.info.paymentUrl.web,
      status: 'PENDING',
      amount: body.amount
    }
  })

  return {
    paymentUrl: linePayResponse.info.paymentUrl.app,
    transactionId: linePayResponse.info.transactionId.toString(),
    orderId: order.id
  }
}
```

### Checkout Page

```typescript
// src/pages/Checkout.tsx
import { useState } from 'react'
import { useNavigate } from 'react-router-dom'
import { Button, Card, List, ListItem, Divider } from '@line/line-ui'
import { PageContainer } from '@/components/layout/PageContainer'
import { useCartStore } from '@/store/cartStore'
import { requestPayment } from '@/services/payment'
import { useUserStore } from '@/store/userStore'
import { AddressSelector } from '@/components/checkout/AddressSelector'
import { PaymentMethodSelector } from '@/components/checkout/PaymentMethodSelector'

export default function Checkout() {
  const navigate = useNavigate()
  const { items, total, clearCart } = useCartStore()
  const { profile } = useUserStore()
  const [selectedAddress, setSelectedAddress] = useState<Address | null>(null)
  const [isProcessing, setIsProcessing] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const deliveryFee = 50
  const grandTotal = total + deliveryFee

  const handleCheckout = async () => {
    if (!selectedAddress) {
      setError('กรุณาเลือกที่อยู่จัดส่ง')
      return
    }

    setIsProcessing(true)
    setError(null)

    try {
      const orderId = `ORD_${Date.now()}`
      
      const result = await requestPayment({
        orderId,
        amount: grandTotal,
        currency: 'THB',
        orderName: `คำสั่งซื้อ ${orderId}`,
        items: items.map(item => ({
          id: item.product.id,
          name: item.product.name,
          imageUrl: item.product.imageUrl,
          quantity: item.quantity,
          price: item.product.price
        }))
      })

      if (result.status === 'success') {
        clearCart()
        navigate(`/orders/${result.orderId}?success=true`)
      } else if (result.status === 'canceled') {
        setError('การชำระเงินถูกยกเลิก')
      } else {
        setError('การชำระเงินล้มเหลว กรุณาลองใหม่')
      }
    } catch (err) {
      setError('เกิดข้อผิดพลาด กรุณาลองใหม่')
    } finally {
      setIsProcessing(false)
    }
  }

  return (
    <PageContainer title="ชำระเงิน" showBack hideNavigation>
      <div className="p-4 space-y-4">
        {/* Order Summary */}
        <Card>
          <div className="p-4">
            <h2 className="font-bold text-gray-800 mb-3">สรุปคำสั่งซื้อ</h2>
            <List>
              {items.map(item => (
                <ListItem
                  key={item.product.id}
                  title={item.product.name}
                  subtitle={`x${item.quantity}`}
                  right={
                    <span className="font-medium">
                      ฿{(item.product.price * item.quantity).toLocaleString()}
                    </span>
                  }
                  thumbnail={
                    <img
                      src={item.product.imageUrl}
                      alt={item.product.name}
                      className="w-12 h-12 object-cover rounded"
                    />
                  }
                />
              ))}
            </List>
            
            <Divider />
            
            <div className="space-y-2 mt-3">
              <div className="flex justify-between text-gray-600">
                <span>ยอดสินค้า</span>
                <span>฿{total.toLocaleString()}</span>
              </div>
              <div className="flex justify-between text-gray-600">
                <span>ค่าจัดส่ง</span>
                <span>฿{deliveryFee}</span>
              </div>
              <Divider />
              <div className="flex justify-between font-bold text-lg">
                <span>รวมทั้งหมด</span>
                <span className="text-green-600">฿{grandTotal.toLocaleString()}</span>
              </div>
            </div>
          </div>
        </Card>

        {/* Address */}
        <AddressSelector
          onSelect={setSelectedAddress}
          selected={selectedAddress}
        />

        {/* Error */}
        {error && (
          <div className="bg-red-50 text-red-600 p-3 rounded-lg text-sm">
            {error}
          </div>
        )}

        {/* Pay Button */}
        <Button
          variant="primary"
          size="large"
          fullWidth
          loading={isProcessing}
          onClick={handleCheckout}
          className="bg-green-500"
        >
          ชำระเงินด้วย LINE Pay ฿{grandTotal.toLocaleString()}
        </Button>
      </div>
    </PageContainer>
  )
}
```

---

## 9. การเผยแพร่ Mini App {#distribution}

### Checklist ก่อน Submit

```
✅ Technical Requirements:
   □ HTTPS endpoint (valid SSL certificate)
   □ Responsive design (320px - 428px width)
   □ LINE UI Kit used for main components
   □ No external login buttons (ใช้ LINE Login เท่านั้น)
   □ Privacy Policy URL
   □ Terms of Service URL
   □ Support email/LINE contact

✅ Content Requirements:
   □ App icon 512x512px (PNG, no transparency)
   □ Screenshots (2-8 ใบ, 1080x1920px)
   □ Short description (ไม่เกิน 100 ตัวอักษร)
   □ Long description (ไม่เกิน 1000 ตัวอักษร)
   □ Category ที่เหมาะสม

✅ Payment Requirements (ถ้ามี):
   □ LINE Pay channel setup
   □ Tax invoice capability
   □ Refund policy stated

✅ Legal Requirements:
   □ Business registration
   □ Age restriction (ถ้าจำเป็น)
   □ Regional compliance
```

### Build สำหรับ Production

```bash
# package.json scripts
{
  "scripts": {
    "dev": "vite --port 3000",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint src --ext ts,tsx",
    "test": "vitest run",
    "test:e2e": "playwright test",
    "analyze": "vite-bundle-visualizer"
  }
}
```

```javascript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { VitePWA } from 'vite-plugin-pwa'
import path from 'path'

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,webp}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\./,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: { maxAgeSeconds: 300 }
            }
          }
        ]
      }
    })
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src')
    }
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          lineui: ['@line/line-ui', '@line/mini-app-sdk'],
          query: ['@tanstack/react-query']
        }
      }
    },
    chunkSizeWarningLimit: 500
  }
})
```

---

## 10. Complete Mini App: Online Store {#online-store}

### Home Page

```typescript
// src/pages/Home.tsx
import { useEffect, useRef } from 'react'
import { useNavigate } from 'react-router-dom'
import { Banner, Card, Button, Chip } from '@line/line-ui'
import { PageContainer } from '@/components/layout/PageContainer'
import { ProductCard } from '@/components/product/ProductCard'
import { useHomeData } from '@/hooks/useHomeData'
import { useAnalytics } from '@/lib/analytics'

export default function Home() {
  const navigate = useNavigate()
  const { banners, categories, featuredProducts, flashSaleProducts } = useHomeData()
  const { trackView } = useAnalytics()
  const viewTracked = useRef(false)

  useEffect(() => {
    if (!viewTracked.current) {
      trackView('home')
      viewTracked.current = true
    }
  }, [])

  return (
    <PageContainer>
      {/* Hero Banners */}
      <div className="w-full">
        <Banner
          items={banners.map(b => ({
            image: b.imageUrl,
            title: b.title,
            onClick: () => navigate(b.linkUrl)
          }))}
          autoPlay
          autoPlayInterval={4000}
          showDots
        />
      </div>

      {/* Categories */}
      <div className="px-4 py-3">
        <h2 className="font-bold text-gray-800 mb-3">หมวดหมู่สินค้า</h2>
        <div className="flex gap-2 overflow-x-auto pb-2 scrollbar-hide">
          {categories.map(cat => (
            <Chip
              key={cat.id}
              label={cat.name}
              icon={<img src={cat.iconUrl} alt="" className="w-4 h-4" />}
              onClick={() => navigate(`/products?category=${cat.id}`)}
              className="whitespace-nowrap flex-shrink-0"
            />
          ))}
        </div>
      </div>

      {/* Flash Sale */}
      {flashSaleProducts.length > 0 && (
        <div className="px-4 py-3">
          <div className="flex justify-between items-center mb-3">
            <h2 className="font-bold text-gray-800">🔥 Flash Sale</h2>
            <FlashSaleTimer />
            <button
              onClick={() => navigate('/products?type=flash-sale')}
              className="text-green-600 text-sm font-medium"
            >
              ดูทั้งหมด
            </button>
          </div>
          <div className="flex gap-3 overflow-x-auto pb-2 scrollbar-hide">
            {flashSaleProducts.slice(0, 6).map(product => (
              <div key={product.id} className="w-40 flex-shrink-0">
                <ProductCard
                  product={product}
                  compact
                  onClick={() => navigate(`/products/${product.id}`)}
                />
              </div>
            ))}
          </div>
        </div>
      )}

      {/* Featured Products */}
      <div className="px-4 py-3">
        <div className="flex justify-between items-center mb-3">
          <h2 className="font-bold text-gray-800">สินค้าแนะนำ</h2>
          <button
            onClick={() => navigate('/products')}
            className="text-green-600 text-sm font-medium"
          >
            ดูทั้งหมด
          </button>
        </div>
        <div className="grid grid-cols-2 gap-3">
          {featuredProducts.slice(0, 6).map(product => (
            <ProductCard
              key={product.id}
              product={product}
              onClick={() => navigate(`/products/${product.id}`)}
            />
          ))}
        </div>
      </div>
    </PageContainer>
  )
}

function FlashSaleTimer() {
  const [timeLeft, setTimeLeft] = useState({ h: 2, m: 30, s: 0 })

  useEffect(() => {
    const timer = setInterval(() => {
      setTimeLeft(prev => {
        if (prev.s > 0) return { ...prev, s: prev.s - 1 }
        if (prev.m > 0) return { ...prev, m: prev.m - 1, s: 59 }
        if (prev.h > 0) return { h: prev.h - 1, m: 59, s: 59 }
        return prev
      })
    }, 1000)
    return () => clearInterval(timer)
  }, [])

  return (
    <div className="flex gap-1 text-sm font-mono">
      <span className="bg-red-500 text-white px-1.5 py-0.5 rounded">
        {String(timeLeft.h).padStart(2, '0')}
      </span>
      <span className="text-red-500 font-bold">:</span>
      <span className="bg-red-500 text-white px-1.5 py-0.5 rounded">
        {String(timeLeft.m).padStart(2, '0')}
      </span>
      <span className="text-red-500 font-bold">:</span>
      <span className="bg-red-500 text-white px-1.5 py-0.5 rounded">
        {String(timeLeft.s).padStart(2, '0')}
      </span>
    </div>
  )
}
```

### Cart Store

```typescript
// src/store/cartStore.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

interface CartItem {
  product: Product
  quantity: number
  selectedVariant?: ProductVariant
}

interface CartStore {
  items: CartItem[]
  total: number
  
  addItem: (product: Product, quantity?: number, variant?: ProductVariant) => void
  removeItem: (productId: string, variantId?: string) => void
  updateQuantity: (productId: string, quantity: number, variantId?: string) => void
  clearCart: () => void
  
  // Computed
  itemCount: number
  totalItems: number
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],
      total: 0,
      
      get itemCount() {
        return get().items.length
      },
      
      get totalItems() {
        return get().items.reduce((sum, item) => sum + item.quantity, 0)
      },

      addItem: (product, quantity = 1, variant) => {
        set(state => {
          const existingIndex = state.items.findIndex(
            item => item.product.id === product.id && 
                    item.selectedVariant?.id === variant?.id
          )

          let newItems: CartItem[]
          
          if (existingIndex >= 0) {
            newItems = state.items.map((item, i) =>
              i === existingIndex
                ? { ...item, quantity: item.quantity + quantity }
                : item
            )
          } else {
            newItems = [...state.items, { product, quantity, selectedVariant: variant }]
          }

          const total = newItems.reduce(
            (sum, item) => sum + (item.selectedVariant?.price ?? item.product.price) * item.quantity,
            0
          )

          return { items: newItems, total }
        })
      },

      removeItem: (productId, variantId) => {
        set(state => {
          const newItems = state.items.filter(
            item => !(item.product.id === productId && 
                      item.selectedVariant?.id === variantId)
          )
          const total = newItems.reduce(
            (sum, item) => sum + (item.selectedVariant?.price ?? item.product.price) * item.quantity,
            0
          )
          return { items: newItems, total }
        })
      },

      updateQuantity: (productId, quantity, variantId) => {
        if (quantity <= 0) {
          get().removeItem(productId, variantId)
          return
        }

        set(state => {
          const newItems = state.items.map(item =>
            item.product.id === productId && item.selectedVariant?.id === variantId
              ? { ...item, quantity }
              : item
          )
          const total = newItems.reduce(
            (sum, item) => sum + (item.selectedVariant?.price ?? item.product.price) * item.quantity,
            0
          )
          return { items: newItems, total }
        })
      },

      clearCart: () => set({ items: [], total: 0 })
    }),
    {
      name: 'cart-storage',
      storage: createJSONStorage(() => localStorage),
      partialize: (state) => ({ items: state.items, total: state.total })
    }
  )
)
```

### Push Notifications Setup

```typescript
// src/services/notifications.ts
import MiniApp from '@line/mini-app-sdk'

interface NotificationPermission {
  granted: boolean
  token?: string
}

export async function requestNotificationPermission(): Promise<NotificationPermission> {
  try {
    const result = await MiniApp.requestPermission({
      permission: 'push_notification'
    })

    if (!result.granted) {
      return { granted: false }
    }

    // ลงทะเบียน push token กับ server
    const token = await MiniApp.getPushToken()
    
    await fetch('/api/users/push-token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${await MiniApp.getAccessToken()}`
      },
      body: JSON.stringify({ token, platform: MiniApp.platform })
    })

    return { granted: true, token }
  } catch (error) {
    console.error('Failed to request notification permission:', error)
    return { granted: false }
  }
}

// ส่ง push notification จาก server
// POST /api/notifications/send
export async function sendPushNotification(params: {
  userId: string
  title: string
  body: string
  data?: Record<string, string>
  imageUrl?: string
}): Promise<void> {
  const response = await fetch('/api/notifications/send', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.SERVICE_TOKEN}`
    },
    body: JSON.stringify(params)
  })

  if (!response.ok) {
    throw new Error('Failed to send notification')
  }
}
```

### Deployment

```yaml
# docker-compose.yml สำหรับ Mini App
version: '3.9'

services:
  mini-app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    environment:
      - VITE_API_URL=https://api.your-domain.com
      - VITE_LINE_CHANNEL_ID=${LINE_CHANNEL_ID}
    restart: unless-stopped

  api:
    build:
      context: ./api
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - LINE_PAY_CHANNEL_ID=${LINE_PAY_CHANNEL_ID}
      - LINE_PAY_CHANNEL_SECRET=${LINE_PAY_CHANNEL_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: miniapp_db
      POSTGRES_USER: miniapp
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

---

### สรุป LINE Mini App

```
✅ ข้อดีของ LINE Mini App:
   - เข้าถึงผู้ใช้ใหม่ผ่าน LINE App Store
   - Native LINE UI ทำให้ UX ดีขึ้น
   - LINE Pay integration ที่ seamless
   - Built-in analytics

⚠️ ข้อควรระวัง:
   - ต้องผ่าน review process
   - ต้องใช้ LINE UI Kit สำหรับ navigation
   - ใช้ได้เฉพาะบางประเทศ (Thailand, Japan, Taiwan)
   - ต้องมี business account

💡 Best Practices:
   - ออกแบบให้ simple และ focused
   - Performance first (< 3s load time)
   - Handle offline gracefully
   - Test บน LINE WebView จริงๆ
   - ใช้ native LINE components เท่าที่ทำได้
```

---

*Part 87 จบแล้ว - ต่อไปใน Part 88: Omnichannel with LINE OA*
