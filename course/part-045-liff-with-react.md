# Part 45: LIFF กับ React - การสร้าง LIFF App ด้วย React

## สารบัญ

1. [Setting up React + LIFF](#setting-up-react-liff)
2. [Create React App + LIFF](#create-react-app-liff)
3. [Vite + React + LIFF](#vite-react-liff)
4. [LIFF Context ใน React](#liff-context-ใน-react)
5. [Custom LIFF Hooks](#custom-liff-hooks)
6. [LIFF กับ React Router](#liff-กับ-react-router)
7. [LIFF กับ Zustand](#liff-กับ-zustand)
8. [Complete Mini-App](#complete-mini-app)
9. [Deployment](#deployment)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Setting up React + LIFF

### 1.1 ทำไมถึงใช้ React กับ LIFF?

```
LIFF + React ช่วยให้:
✅ Component-based Architecture
✅ State Management ที่ดีกว่า Vanilla JS
✅ Reusable LIFF Hooks
✅ Type Safety ด้วย TypeScript
✅ Rich Ecosystem (Router, State Mgmt, UI Libraries)
✅ Developer Experience ที่ดีกว่า

เหมาะสำหรับ:
- LIFF App ขนาดกลาง-ใหญ่
- หลาย Component ที่ใช้ LIFF Data ร่วมกัน
- ทีม Developer ที่คุ้นเคยกับ React
```

### 1.2 เปรียบเทียบ Options

```
Create React App    Vite + React       Next.js
─────────────────   ──────────────     ────────
Simple Setup        Fast Build         SSR Support
Slower Build        Modern             Complex Setup
Zero Config         Recommended        Overkill for LIFF
Familiar            TypeScript Ready   Good for SEO
```

---

## 2. Create React App + LIFF

### 2.1 สร้างโปรเจกต์

```bash
# สร้าง React App
npx create-react-app my-liff-app
cd my-liff-app

# ติดตั้ง LIFF SDK
npm install @line/liff

# ติดตั้ง Dependencies อื่นๆ
npm install react-router-dom axios
```

### 2.2 โครงสร้างโปรเจกต์

```
my-liff-app/
├── public/
│   └── index.html
├── src/
│   ├── components/
│   │   ├── common/
│   │   │   ├── LoadingSpinner.jsx
│   │   │   ├── ErrorBoundary.jsx
│   │   │   └── Toast.jsx
│   │   ├── profile/
│   │   │   └── ProfileCard.jsx
│   │   ├── messages/
│   │   │   ├── MessageForm.jsx
│   │   │   └── ShareButton.jsx
│   │   └── scanner/
│   │       └── QRScanner.jsx
│   ├── contexts/
│   │   └── LiffContext.jsx       ← LIFF Context Provider
│   ├── hooks/
│   │   ├── useLiff.js            ← Main LIFF Hook
│   │   ├── useProfile.js         ← Profile Hook
│   │   ├── useMessages.js        ← Messages Hook
│   │   └── useQRScanner.js       ← QR Scanner Hook
│   ├── pages/
│   │   ├── Home.jsx
│   │   ├── Profile.jsx
│   │   ├── Messages.jsx
│   │   └── Scanner.jsx
│   ├── services/
│   │   └── api.js               ← API calls
│   ├── store/
│   │   └── useLiffStore.js      ← Zustand Store
│   ├── utils/
│   │   └── liff.js              ← LIFF Utilities
│   ├── App.jsx
│   ├── index.js
│   └── styles.css
├── .env
├── .env.development
└── package.json
```

### 2.3 Environment Variables

```bash
# .env
REACT_APP_LIFF_ID=1234567890-AbcdEfgh
REACT_APP_API_BASE_URL=https://your-api.com/api

# .env.development
REACT_APP_LIFF_ID=1234567890-DevLiffId
REACT_APP_API_BASE_URL=http://localhost:3001/api
```

---

## 3. Vite + React + LIFF

### 3.1 สร้างโปรเจกต์ด้วย Vite

```bash
# สร้าง Vite + React + TypeScript
npm create vite@latest my-liff-app -- --template react-ts
cd my-liff-app

# ติดตั้ง Dependencies
npm install
npm install @line/liff
npm install react-router-dom axios zustand
npm install -D @types/react @types/react-dom

# รัน Development Server
npm run dev
```

### 3.2 Vite Configuration

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
    plugins: [react()],
    
    resolve: {
        alias: {
            '@': path.resolve(__dirname, './src')
        }
    },
    
    server: {
        port: 3000,
        // สำหรับ HTTPS Local Dev (ต้องการสำหรับ LIFF บางครั้ง)
        // https: true
    },
    
    define: {
        // ทำให้ process.env ใช้งานได้
        'process.env': {}
    }
});
```

### 3.3 TypeScript Config สำหรับ LIFF

```typescript
// src/types/liff.d.ts
import { Liff } from '@line/liff';

// Extend Window Interface
declare global {
    interface Window {
        liff: Liff;
    }
}

// LIFF Profile Type
export interface LiffProfile {
    userId: string;
    displayName: string;
    pictureUrl: string;
    statusMessage?: string;
}

// LIFF Context Type
export interface LiffContextType {
    type: string;
    viewType: string;
    userId?: string;
}

// App State
export interface LiffAppState {
    initialized: boolean;
    loggedIn: boolean;
    profile: LiffProfile | null;
    error: string | null;
    loading: boolean;
}
```

---

## 4. LIFF Context ใน React

### 4.1 LiffContext Provider

```jsx
// src/contexts/LiffContext.jsx
import { createContext, useContext, useState, useEffect, useCallback } from 'react';
import liff from '@line/liff';

// Create Context
const LiffContext = createContext(null);

// Context Value Type
// {
//   initialized: boolean
//   loggedIn: boolean
//   profile: object | null
//   loading: boolean
//   error: string | null
//   login: Function
//   logout: Function
//   sendMessages: Function
//   shareMessages: Function
// }

// Provider Component
export function LiffProvider({ children, liffId }) {
    const [state, setState] = useState({
        initialized: false,
        loggedIn: false,
        profile: null,
        loading: true,
        error: null
    });
    
    // Initialize LIFF
    useEffect(() => {
        async function initializeLiff() {
            try {
                await liff.init({ liffId });
                
                let profile = null;
                
                if (liff.isLoggedIn()) {
                    try {
                        profile = await liff.getProfile();
                    } catch (profileError) {
                        console.warn('Failed to get profile:', profileError);
                    }
                }
                
                setState({
                    initialized: true,
                    loggedIn: liff.isLoggedIn(),
                    profile,
                    loading: false,
                    error: null
                });
                
            } catch (error) {
                console.error('LIFF init failed:', error);
                setState(prev => ({
                    ...prev,
                    initialized: false,
                    loading: false,
                    error: error.message || 'LIFF initialization failed'
                }));
            }
        }
        
        if (liffId) {
            initializeLiff();
        } else {
            setState(prev => ({
                ...prev,
                loading: false,
                error: 'LIFF ID is required'
            }));
        }
    }, [liffId]);
    
    // Login
    const login = useCallback((options = {}) => {
        liff.login({
            redirectUri: window.location.href,
            ...options
        });
    }, []);
    
    // Logout
    const logout = useCallback(() => {
        liff.logout();
        setState(prev => ({
            ...prev,
            loggedIn: false,
            profile: null
        }));
        window.location.reload();
    }, []);
    
    // Refresh Profile
    const refreshProfile = useCallback(async () => {
        if (!liff.isLoggedIn()) return null;
        
        try {
            const profile = await liff.getProfile();
            setState(prev => ({ ...prev, profile }));
            return profile;
        } catch (error) {
            console.error('Refresh profile error:', error);
            return null;
        }
    }, []);
    
    // Send Messages
    const sendMessages = useCallback(async (messages) => {
        if (!liff.isInClient()) {
            throw new Error('Must be opened in LINE App');
        }
        
        if (!liff.isLoggedIn()) {
            throw new Error('Must be logged in');
        }
        
        return await liff.sendMessages(messages);
    }, []);
    
    // Share Messages
    const shareMessages = useCallback(async (messages, options = {}) => {
        if (!liff.isInClient()) {
            throw new Error('Must be opened in LINE App');
        }
        
        return await liff.shareTargetPicker(messages, options);
    }, []);
    
    // Context Value
    const contextValue = {
        // State
        ...state,
        
        // LIFF Info
        isInClient: liff.isInClient?.() ?? false,
        os: liff.getOS?.() ?? 'web',
        language: liff.getLanguage?.() ?? 'th',
        version: liff.getVersion?.() ?? '0.0.0',
        context: liff.getContext?.() ?? null,
        
        // Actions
        login,
        logout,
        refreshProfile,
        sendMessages,
        shareMessages,
        
        // Raw LIFF instance (for advanced usage)
        liff
    };
    
    return (
        <LiffContext.Provider value={contextValue}>
            {children}
        </LiffContext.Provider>
    );
}

// Hook
export function useLiffContext() {
    const context = useContext(LiffContext);
    if (!context) {
        throw new Error('useLiffContext must be used within LiffProvider');
    }
    return context;
}

export default LiffContext;
```

### 4.2 App.jsx - ใช้ LiffProvider

```jsx
// src/App.jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { LiffProvider } from './contexts/LiffContext';
import Home from './pages/Home';
import Profile from './pages/Profile';
import Messages from './pages/Messages';
import Scanner from './pages/Scanner';
import LoadingScreen from './components/common/LoadingScreen';
import ErrorScreen from './components/common/ErrorScreen';
import { useLiffContext } from './contexts/LiffContext';

const LIFF_ID = import.meta.env.VITE_LIFF_ID || process.env.REACT_APP_LIFF_ID;

// Inner App (has access to LiffContext)
function AppContent() {
    const { loading, error, initialized } = useLiffContext();
    
    if (loading) {
        return <LoadingScreen message="กำลังโหลด LIFF..." />;
    }
    
    if (error) {
        return <ErrorScreen message={error} />;
    }
    
    if (!initialized) {
        return <ErrorScreen message="ไม่สามารถเริ่ม LIFF ได้" />;
    }
    
    return (
        <Routes>
            <Route path="/" element={<Home />} />
            <Route path="/profile" element={<Profile />} />
            <Route path="/messages" element={<Messages />} />
            <Route path="/scanner" element={<Scanner />} />
        </Routes>
    );
}

// Root App
function App() {
    return (
        <LiffProvider liffId={LIFF_ID}>
            <BrowserRouter>
                <AppContent />
            </BrowserRouter>
        </LiffProvider>
    );
}

export default App;
```

---

## 5. Custom LIFF Hooks

### 5.1 useLiff() - Main Hook

```javascript
// src/hooks/useLiff.js
import { useLiffContext } from '../contexts/LiffContext';

/**
 * Main LIFF Hook - ใช้ LIFF State และ Actions
 */
export function useLiff() {
    const liffContext = useLiffContext();
    return liffContext;
}

// ตัวอย่างการใช้งาน
/*
function MyComponent() {
    const { initialized, loggedIn, profile, login, logout } = useLiff();
    
    if (!loggedIn) {
        return <button onClick={login}>Login</button>;
    }
    
    return <div>Hello, {profile?.displayName}!</div>;
}
*/
```

### 5.2 useProfile() - Profile Hook

```javascript
// src/hooks/useProfile.js
import { useState, useCallback } from 'react';
import { useLiffContext } from '../contexts/LiffContext';

/**
 * Hook สำหรับจัดการข้อมูล Profile
 */
export function useProfile() {
    const { loggedIn, profile, refreshProfile, login, logout } = useLiffContext();
    const [updating, setUpdating] = useState(false);
    
    // Refresh Profile
    const refresh = useCallback(async () => {
        if (!loggedIn) return null;
        setUpdating(true);
        try {
            return await refreshProfile();
        } finally {
            setUpdating(false);
        }
    }, [loggedIn, refreshProfile]);
    
    return {
        profile,
        loggedIn,
        updating,
        
        // Actions
        login,
        logout,
        refresh,
        
        // Computed
        displayName: profile?.displayName ?? ''),
        pictureUrl: profile?.pictureUrl ?? null,
        userId: profile?.userId ?? null,
        statusMessage: profile?.statusMessage ?? null,
        
        // Guards
        requireLogin: !loggedIn
    };
}
```

### 5.3 useMessages() - Messages Hook

```javascript
// src/hooks/useMessages.js
import { useState, useCallback } from 'react';
import { useLiffContext } from '../contexts/LiffContext';

/**
 * Hook สำหรับส่งข้อความ
 */
export function useMessages() {
    const { sendMessages: liffSend, shareMessages: liffShare, isInClient } = useLiffContext();
    
    const [sending, setSending] = useState(false);
    const [sharing, setSharing] = useState(false);
    const [lastError, setLastError] = useState(null);
    const [lastSentAt, setLastSentAt] = useState(null);
    
    // Send to Current Chat
    const send = useCallback(async (messages) => {
        if (!isInClient) {
            throw new Error('ต้องเปิดจาก LINE App');
        }
        
        setSending(true);
        setLastError(null);
        
        try {
            // Normalize messages to array
            const msgArray = Array.isArray(messages) ? messages : [messages];
            await liffSend(msgArray);
            setLastSentAt(new Date());
            return true;
        } catch (error) {
            setLastError(error.message);
            throw error;
        } finally {
            setSending(false);
        }
    }, [liffSend, isInClient]);
    
    // Send Text Message
    const sendText = useCallback(async (text) => {
        return await send({ type: 'text', text });
    }, [send]);
    
    // Send Flex Message
    const sendFlex = useCallback(async (altText, contents) => {
        return await send({
            type: 'flex',
            altText,
            contents
        });
    }, [send]);
    
    // Share to Other Chats
    const share = useCallback(async (messages, options = {}) => {
        if (!isInClient) {
            throw new Error('ต้องเปิดจาก LINE App');
        }
        
        setSharing(true);
        setLastError(null);
        
        try {
            const msgArray = Array.isArray(messages) ? messages : [messages];
            const result = await liffShare(msgArray, options);
            return result !== null; // null = cancelled
        } catch (error) {
            if (error.code !== 'CANCEL') {
                setLastError(error.message);
                throw error;
            }
            return false;
        } finally {
            setSharing(false);
        }
    }, [liffShare, isInClient]);
    
    // Share Text
    const shareText = useCallback(async (text, isMultiple = false) => {
        return await share(
            { type: 'text', text },
            { isMultiple }
        );
    }, [share]);
    
    return {
        // State
        sending,
        sharing,
        lastError,
        lastSentAt,
        canSend: isInClient,
        
        // Actions
        send,
        sendText,
        sendFlex,
        share,
        shareText,
        
        // Clear Error
        clearError: () => setLastError(null)
    };
}
```

### 5.4 useQRScanner() - QR Scanner Hook

```javascript
// src/hooks/useQRScanner.js
import { useState, useCallback } from 'react';
import liff from '@line/liff';

/**
 * Hook สำหรับ QR Code Scanner
 */
export function useQRScanner() {
    const [scanning, setScanning] = useState(false);
    const [lastResult, setLastResult] = useState(null);
    const [history, setHistory] = useState([]);
    const [error, setError] = useState(null);
    
    const scan = useCallback(async () => {
        if (!liff.isInClient()) {
            setError('QR Scanner ใช้ได้เฉพาะใน LINE App');
            return null;
        }
        
        setScanning(true);
        setError(null);
        
        try {
            const result = await liff.scanCodeV2();
            
            if (result?.value) {
                const scanItem = {
                    value: result.value,
                    timestamp: new Date(),
                    id: Date.now()
                };
                
                setLastResult(scanItem);
                setHistory(prev => [scanItem, ...prev.slice(0, 19)]); // Keep last 20
                
                return result.value;
            }
            
            return null;
        } catch (err) {
            if (err.code === 'CANCEL') {
                return null; // User cancelled
            }
            setError(err.message);
            return null;
        } finally {
            setScanning(false);
        }
    }, []);
    
    const clearHistory = useCallback(() => {
        setHistory([]);
        setLastResult(null);
    }, []);
    
    const clearError = useCallback(() => setError(null), []);
    
    return {
        scanning,
        lastResult,
        history,
        error,
        scan,
        clearHistory,
        clearError,
        canScan: liff.isInClient?.() ?? false
    };
}
```

### 5.5 useLiffInit() - Initialization Hook

```javascript
// src/hooks/useLiffInit.js
import { useState, useEffect } from 'react';
import liff from '@line/liff';

/**
 * Hook สำหรับ Initialize LIFF แบบ Manual
 * ใช้เมื่อไม่ต้องการ LiffProvider
 */
export function useLiffInit(liffId) {
    const [state, setState] = useState({
        initialized: false,
        loading: true,
        error: null
    });
    
    useEffect(() => {
        if (!liffId) {
            setState({ initialized: false, loading: false, error: 'LIFF ID required' });
            return;
        }
        
        liff.init({ liffId })
            .then(() => {
                setState({ initialized: true, loading: false, error: null });
            })
            .catch(error => {
                setState({
                    initialized: false,
                    loading: false,
                    error: error.message
                });
            });
    }, [liffId]);
    
    return state;
}
```

---

## 6. LIFF กับ React Router

### 6.1 Setup React Router สำหรับ LIFF

```jsx
// src/App.jsx - React Router Setup
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { useLiffContext } from './contexts/LiffContext';

// Protected Route Component
function ProtectedRoute({ element }) {
    const { loggedIn, loading, login } = useLiffContext();
    
    if (loading) return null;
    
    if (!loggedIn) {
        // Auto login
        login();
        return <div>กำลัง Login...</div>;
    }
    
    return element;
}

// Layout Component
function Layout({ children }) {
    return (
        <div className="layout">
            <main className="content">{children}</main>
            <BottomNav />
        </div>
    );
}

// Bottom Navigation
function BottomNav() {
    const location = useLocation();
    
    const navItems = [
        { path: '/', icon: '🏠', label: 'หน้าหลัก' },
        { path: '/products', icon: '🛍️', label: 'สินค้า' },
        { path: '/scanner', icon: '📷', label: 'สแกน' },
        { path: '/profile', icon: '👤', label: 'โปรไฟล์' }
    ];
    
    return (
        <nav style={{
            position: 'fixed',
            bottom: 0,
            left: 0,
            right: 0,
            display: 'flex',
            background: 'white',
            borderTop: '1px solid #e5e7eb',
            paddingBottom: 'env(safe-area-inset-bottom)'
        }}>
            {navItems.map(item => (
                <NavLink
                    key={item.path}
                    to={item.path}
                    style={({ isActive }) => ({
                        flex: 1,
                        display: 'flex',
                        flexDirection: 'column',
                        alignItems: 'center',
                        padding: '10px 4px',
                        textDecoration: 'none',
                        color: isActive ? '#06c755' : '#9ca3af',
                        fontSize: '10px',
                        gap: '4px',
                        fontWeight: isActive ? '700' : '400'
                    })}
                >
                    <span style={{ fontSize: '20px' }}>{item.icon}</span>
                    {item.label}
                </NavLink>
            ))}
        </nav>
    );
}
```

### 6.2 LIFF Deep Link Routing

```jsx
// src/hooks/useLiffDeepLink.js
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

/**
 * Handle LIFF Deep Links
 * URL: https://liff.line.me/LIFF_ID/products?category=clothes
 * App Path: /products?category=clothes
 */
export function useLiffDeepLink() {
    const navigate = useNavigate();
    
    useEffect(() => {
        // LIFF SDK ตัดพาร์ท LIFF ID ออกไปแล้ว
        // window.location.pathname จะเป็น Path ของแอป
        const path = window.location.pathname;
        const search = window.location.search;
        
        if (path !== '/' && path !== '') {
            navigate(path + search, { replace: true });
        }
    }, [navigate]);
}
```

---

## 7. LIFF กับ Zustand

### 7.1 LIFF Store

```javascript
// src/store/useLiffStore.js
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';
import liff from '@line/liff';

const useLiffStore = create(
    devtools(
        (set, get) => ({
            // State
            initialized: false,
            loading: true,
            error: null,
            profile: null,
            loggedIn: false,
            
            // LIFF Info
            isInClient: false,
            os: 'web',
            language: 'th',
            version: '0.0.0',
            context: null,
            
            // Actions
            init: async (liffId) => {
                set({ loading: true, error: null });
                
                try {
                    await liff.init({ liffId });
                    
                    const info = {
                        initialized: true,
                        loading: false,
                        loggedIn: liff.isLoggedIn(),
                        isInClient: liff.isInClient(),
                        os: liff.getOS(),
                        language: liff.getLanguage(),
                        version: liff.getVersion(),
                        context: liff.getContext()
                    };
                    
                    if (liff.isLoggedIn()) {
                        try {
                            const profile = await liff.getProfile();
                            info.profile = profile;
                        } catch (e) {
                            console.warn('Profile fetch failed:', e);
                        }
                    }
                    
                    set(info);
                    
                } catch (error) {
                    set({
                        initialized: false,
                        loading: false,
                        error: error.message
                    });
                }
            },
            
            login: () => {
                liff.login({ redirectUri: window.location.href });
            },
            
            logout: () => {
                liff.logout();
                set({ loggedIn: false, profile: null });
                window.location.reload();
            },
            
            refreshProfile: async () => {
                if (!liff.isLoggedIn()) return null;
                
                try {
                    const profile = await liff.getProfile();
                    set({ profile });
                    return profile;
                } catch (error) {
                    console.error('Refresh profile error:', error);
                    return null;
                }
            },
            
            sendMessages: async (messages) => {
                if (!liff.isInClient()) {
                    throw new Error('Must be in LINE App');
                }
                await liff.sendMessages(messages);
            },
            
            shareMessages: async (messages, options = {}) => {
                if (!liff.isInClient()) {
                    throw new Error('Must be in LINE App');
                }
                return await liff.shareTargetPicker(messages, options);
            }
        }),
        { name: 'liff-store' }
    )
);

export default useLiffStore;
```

### 7.2 Cart Store (สำหรับ Mini Shopping App)

```javascript
// src/store/useCartStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

const useCartStore = create(
    persist(
        (set, get) => ({
            items: [],
            
            addItem: (product, quantity = 1) => {
                set(state => {
                    const existing = state.items.find(i => i.id === product.id);
                    
                    if (existing) {
                        return {
                            items: state.items.map(item =>
                                item.id === product.id
                                    ? { ...item, quantity: item.quantity + quantity }
                                    : item
                            )
                        };
                    }
                    
                    return {
                        items: [...state.items, { ...product, quantity }]
                    };
                });
            },
            
            removeItem: (productId) => {
                set(state => ({
                    items: state.items.filter(i => i.id !== productId)
                }));
            },
            
            updateQuantity: (productId, quantity) => {
                if (quantity <= 0) {
                    get().removeItem(productId);
                    return;
                }
                
                set(state => ({
                    items: state.items.map(item =>
                        item.id === productId
                            ? { ...item, quantity }
                            : item
                    )
                }));
            },
            
            clearCart: () => set({ items: [] }),
            
            // Computed
            getTotalItems: () => {
                return get().items.reduce((sum, item) => sum + item.quantity, 0);
            },
            
            getTotalPrice: () => {
                return get().items.reduce(
                    (sum, item) => sum + (item.price * item.quantity), 
                    0
                );
            },
            
            getItemCount: (productId) => {
                return get().items.find(i => i.id === productId)?.quantity ?? 0;
            }
        }),
        {
            name: 'liff-cart',
            getStorage: () => sessionStorage
        }
    )
);

export default useCartStore;
```

---

## 8. Complete Mini-App

### 8.1 Main App Structure

```jsx
// src/App.jsx - Complete Mini App
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Suspense, lazy } from 'react';
import { LiffProvider } from './contexts/LiffContext';
import AppShell from './components/layout/AppShell';
import LoadingScreen from './components/common/LoadingScreen';
import InitGuard from './components/common/InitGuard';

// Lazy Load Pages
const HomePage = lazy(() => import('./pages/Home'));
const ProfilePage = lazy(() => import('./pages/Profile'));
const MessagesPage = lazy(() => import('./pages/Messages'));
const ScannerPage = lazy(() => import('./pages/Scanner'));
const CartPage = lazy(() => import('./pages/Cart'));

const LIFF_ID = import.meta.env.VITE_LIFF_ID;

export default function App() {
    return (
        <LiffProvider liffId={LIFF_ID}>
            <BrowserRouter>
                <InitGuard>
                    <AppShell>
                        <Suspense fallback={<LoadingScreen />}>
                            <Routes>
                                <Route path="/" element={<HomePage />} />
                                <Route path="/profile" element={<ProfilePage />} />
                                <Route path="/messages" element={<MessagesPage />} />
                                <Route path="/scanner" element={<ScannerPage />} />
                                <Route path="/cart" element={<CartPage />} />
                            </Routes>
                        </Suspense>
                    </AppShell>
                </InitGuard>
            </BrowserRouter>
        </LiffProvider>
    );
}
```

### 8.2 Profile Page Component

```jsx
// src/pages/Profile.jsx
import { useState } from 'react';
import { useProfile } from '../hooks/useProfile';
import { useMessages } from '../hooks/useMessages';

export default function ProfilePage() {
    const { profile, loggedIn, loading: profileLoading, login, logout, refresh } = useProfile();
    const { sendText, sending } = useMessages();
    const [refreshing, setRefreshing] = useState(false);
    
    const handleRefresh = async () => {
        setRefreshing(true);
        await refresh();
        setRefreshing(false);
    };
    
    const handleSendProfile = async () => {
        if (!profile) return;
        
        const text = `👤 ข้อมูลโปรไฟล์\n━━━━━━━━━━━\nชื่อ: ${profile.displayName}\nID: ${profile.userId}`;
        
        try {
            await sendText(text);
            alert('ส่งข้อมูลใน Chat แล้ว!');
        } catch (error) {
            if (!error.message.includes('LINE App')) {
                alert('เกิดข้อผิดพลาด: ' + error.message);
            }
        }
    };
    
    if (!loggedIn) {
        return (
            <div style={{ display: 'flex', flexDirection: 'column', alignItems: 'center',
                         justifyContent: 'center', minHeight: '60vh', gap: 16, padding: 32,
                         textAlign: 'center' }}>
                <div style={{ fontSize: 64 }}>🔒</div>
                <h2>กรุณา Login</h2>
                <p style={{ color: '#666' }}>Login เพื่อดูข้อมูลโปรไฟล์ของคุณ</p>
                <button
                    onClick={login}
                    style={{ padding: '14px 32px', background: '#06c755', color: 'white',
                             border: 'none', borderRadius: 12, fontSize: 16, fontWeight: 700,
                             cursor: 'pointer' }}
                >
                    Login ด้วย LINE
                </button>
            </div>
        );
    }
    
    return (
        <div style={{ padding: '0 0 80px' }}>
            {/* Profile Header */}
            <div style={{ background: 'linear-gradient(135deg, #06c755, #04a040)',
                         padding: '32px 20px 60px', textAlign: 'center', color: 'white' }}>
                <h1 style={{ fontSize: 20, fontWeight: 700, margin: 0 }}>โปรไฟล์ของฉัน</h1>
            </div>
            
            {/* Profile Card */}
            <div style={{ margin: '-40px 16px 0', background: 'white', borderRadius: 20,
                         padding: 24, boxShadow: '0 8px 24px rgba(0,0,0,0.12)',
                         textAlign: 'center', position: 'relative', zIndex: 1 }}>
                
                {profile?.pictureUrl ? (
                    <img
                        src={profile.pictureUrl}
                        alt="Profile"
                        style={{ width: 96, height: 96, borderRadius: '50%',
                                border: '4px solid #06c755', objectFit: 'cover',
                                marginBottom: 16 }}
                    />
                ) : (
                    <div style={{ width: 96, height: 96, borderRadius: '50%',
                                background: '#e5e7eb', display: 'flex',
                                alignItems: 'center', justifyContent: 'center',
                                fontSize: 40, margin: '0 auto 16px' }}>
                        👤
                    </div>
                )}
                
                <h2 style={{ fontSize: 22, fontWeight: 700, margin: '0 0 4px' }}>
                    {profile?.displayName || '-'}
                </h2>
                
                {profile?.statusMessage && (
                    <p style={{ color: '#666', fontStyle: 'italic', margin: '0 0 16px', fontSize: 14 }}>
                        "{profile.statusMessage}"
                    </p>
                )}
                
                <div style={{ background: '#f3f4f6', borderRadius: 10, padding: 12,
                             textAlign: 'left' }}>
                    <div style={{ fontSize: 10, fontWeight: 700, textTransform: 'uppercase',
                                color: '#9ca3af', marginBottom: 4 }}>LINE User ID</div>
                    <div style={{ fontSize: 12, fontFamily: 'monospace', 
                                color: '#374151', wordBreak: 'break-all' }}>
                        {profile?.userId || '-'}
                    </div>
                </div>
            </div>
            
            {/* Action Buttons */}
            <div style={{ margin: '16px 16px 0', display: 'flex', flexDirection: 'column', gap: 12 }}>
                <button
                    onClick={handleSendProfile}
                    disabled={sending}
                    style={{ padding: '14px', background: sending ? '#9ca3af' : '#06c755',
                             color: 'white', border: 'none', borderRadius: 12,
                             fontSize: 15, fontWeight: 700, cursor: sending ? 'not-allowed' : 'pointer' }}
                >
                    {sending ? 'กำลังส่ง...' : '💬 ส่งข้อมูลใน Chat'}
                </button>
                
                <button
                    onClick={handleRefresh}
                    disabled={refreshing}
                    style={{ padding: '14px', background: 'white',
                             color: '#374151', border: '1.5px solid #d1d5db',
                             borderRadius: 12, fontSize: 15, fontWeight: 600,
                             cursor: 'pointer' }}
                >
                    {refreshing ? 'กำลังรีเฟรช...' : '🔄 รีเฟรชโปรไฟล์'}
                </button>
                
                <button
                    onClick={() => {
                        if (window.confirm('ต้องการ Logout หรือไม่?')) {
                            logout();
                        }
                    }}
                    style={{ padding: '14px', background: 'white',
                             color: '#dc2626', border: '1.5px solid #fca5a5',
                             borderRadius: 12, fontSize: 15, fontWeight: 600,
                             cursor: 'pointer' }}
                >
                    🚪 Logout
                </button>
            </div>
        </div>
    );
}
```

### 8.3 Messages Page Component

```jsx
// src/pages/Messages.jsx
import { useState } from 'react';
import { useMessages } from '../hooks/useMessages';
import { useLiffContext } from '../contexts/LiffContext';

const MESSAGE_TEMPLATES = [
    {
        id: 'greeting',
        label: 'ทักทาย',
        icon: '👋',
        text: 'สวัสดีครับ/ค่ะ!\nต้องการข้อมูลเพิ่มเติมไหมครับ?'
    },
    {
        id: 'order',
        label: 'สั่งซื้อ',
        icon: '🛒',
        text: 'ฉันต้องการสั่งซื้อสินค้า'
    },
    {
        id: 'support',
        label: 'ขอความช่วยเหลือ',
        icon: '🆘',
        text: 'ต้องการความช่วยเหลือ กรุณาติดต่อกลับด้วยนะครับ/ค่ะ'
    },
    {
        id: 'custom',
        label: 'ข้อความเอง',
        icon: '✏️',
        text: null
    }
];

export default function MessagesPage() {
    const { isInClient } = useLiffContext();
    const { sendText, sendFlex, share, sending, sharing, lastError, clearError } = useMessages();
    
    const [selectedTemplate, setSelectedTemplate] = useState(null);
    const [customText, setCustomText] = useState('');
    const [feedback, setFeedback] = useState(null);
    
    const handleSend = async () => {
        const template = MESSAGE_TEMPLATES.find(t => t.id === selectedTemplate);
        const text = template?.id === 'custom' ? customText : template?.text;
        
        if (!text?.trim()) {
            alert('กรุณาเลือก Template หรือพิมพ์ข้อความ');
            return;
        }
        
        try {
            await sendText(text);
            setFeedback({ type: 'success', message: '✅ ส่งข้อความแล้ว!' });
            setTimeout(() => {
                setFeedback(null);
                // ปิด LIFF หลัง 1.5 วิ
                // liff.closeWindow();
            }, 1500);
        } catch (error) {
            setFeedback({ type: 'error', message: '❌ ' + error.message });
        }
    };
    
    const handleShare = async () => {
        const template = MESSAGE_TEMPLATES.find(t => t.id === selectedTemplate);
        const text = template?.id === 'custom' ? customText : template?.text;
        
        if (!text?.trim()) return;
        
        try {
            const shared = await share({ type: 'text', text }, { isMultiple: true });
            if (shared) {
                setFeedback({ type: 'success', message: '✅ แชร์สำเร็จ!' });
            }
        } catch (error) {
            setFeedback({ type: 'error', message: '❌ ' + error.message });
        }
    };
    
    return (
        <div style={{ padding: '24px 16px 100px' }}>
            <h2 style={{ fontSize: 20, fontWeight: 700, marginBottom: 24 }}>ส่งข้อความ</h2>
            
            {/* Not In Client Warning */}
            {!isInClient && (
                <div style={{ background: '#fff3cd', border: '1px solid #ffc107',
                             borderRadius: 10, padding: '12px 16px', marginBottom: 16,
                             fontSize: 13, color: '#856404' }}>
                    ⚠️ บางฟีเจอร์ใช้ได้เฉพาะใน LINE App
                </div>
            )}
            
            {/* Template Selector */}
            <div style={{ marginBottom: 24 }}>
                <label style={{ display: 'block', fontSize: 13, fontWeight: 700,
                               color: '#374151', marginBottom: 12,
                               textTransform: 'uppercase', letterSpacing: '0.05em' }}>
                    เลือก Template
                </label>
                <div style={{ display: 'grid', gridTemplateColumns: 'repeat(2, 1fr)', gap: 10 }}>
                    {MESSAGE_TEMPLATES.map(template => (
                        <button
                            key={template.id}
                            onClick={() => setSelectedTemplate(template.id)}
                            style={{
                                padding: '14px 12px',
                                borderRadius: 12,
                                border: selectedTemplate === template.id
                                    ? '2px solid #06c755'
                                    : '1.5px solid #e5e7eb',
                                background: selectedTemplate === template.id
                                    ? '#f0fdf4' : 'white',
                                cursor: 'pointer',
                                textAlign: 'left',
                                display: 'flex',
                                alignItems: 'center',
                                gap: 8
                            }}
                        >
                            <span style={{ fontSize: 22 }}>{template.icon}</span>
                            <span style={{
                                fontWeight: 600,
                                fontSize: 13,
                                color: selectedTemplate === template.id ? '#166534' : '#374151'
                            }}>
                                {template.label}
                            </span>
                        </button>
                    ))}
                </div>
            </div>
            
            {/* Custom Text Input */}
            {selectedTemplate === 'custom' && (
                <div style={{ marginBottom: 24 }}>
                    <label style={{ display: 'block', fontSize: 13, fontWeight: 700,
                                   color: '#374151', marginBottom: 8 }}>
                        ข้อความของคุณ
                    </label>
                    <textarea
                        value={customText}
                        onChange={e => setCustomText(e.target.value)}
                        placeholder="พิมพ์ข้อความที่ต้องการส่ง..."
                        style={{ width: '100%', padding: 12, border: '1.5px solid #e5e7eb',
                                borderRadius: 10, fontSize: 15, height: 100,
                                resize: 'none', outline: 'none', boxSizing: 'border-box' }}
                    />
                    <div style={{ textAlign: 'right', fontSize: 12, color: '#9ca3af', marginTop: 4 }}>
                        {customText.length}/500
                    </div>
                </div>
            )}
            
            {/* Preview */}
            {selectedTemplate && selectedTemplate !== 'custom' && (
                <div style={{ background: '#f8f9fa', borderRadius: 12, padding: 16, marginBottom: 24 }}>
                    <div style={{ fontSize: 12, color: '#9ca3af', marginBottom: 8,
                                fontWeight: 700, textTransform: 'uppercase' }}>
                        ตัวอย่างข้อความ
                    </div>
                    <div style={{ fontSize: 14, color: '#374151', whiteSpace: 'pre-wrap',
                                lineHeight: 1.6 }}>
                        {MESSAGE_TEMPLATES.find(t => t.id === selectedTemplate)?.text}
                    </div>
                </div>
            )}
            
            {/* Feedback */}
            {feedback && (
                <div style={{
                    padding: '12px 16px',
                    borderRadius: 10,
                    marginBottom: 16,
                    background: feedback.type === 'success' ? '#dcfce7' : '#fee2e2',
                    color: feedback.type === 'success' ? '#166534' : '#dc2626',
                    fontWeight: 600,
                    fontSize: 14
                }}>
                    {feedback.message}
                </div>
            )}
            
            {/* Action Buttons */}
            <div style={{ display: 'flex', gap: 12 }}>
                <button
                    onClick={handleShare}
                    disabled={sharing || !selectedTemplate}
                    style={{
                        flex: 1,
                        padding: '14px',
                        background: 'white',
                        color: '#06c755',
                        border: '2px solid #06c755',
                        borderRadius: 12,
                        fontSize: 14,
                        fontWeight: 700,
                        cursor: sharing || !selectedTemplate ? 'not-allowed' : 'pointer',
                        opacity: !selectedTemplate ? 0.5 : 1
                    }}
                >
                    {sharing ? '...' : '📤 แชร์'}
                </button>
                
                <button
                    onClick={handleSend}
                    disabled={sending || !selectedTemplate || !isInClient}
                    style={{
                        flex: 2,
                        padding: '14px',
                        background: !selectedTemplate || !isInClient || sending
                            ? '#9ca3af' : '#06c755',
                        color: 'white',
                        border: 'none',
                        borderRadius: 12,
                        fontSize: 15,
                        fontWeight: 700,
                        cursor: sending || !selectedTemplate || !isInClient
                            ? 'not-allowed' : 'pointer'
                    }}
                >
                    {sending ? 'กำลังส่ง...' : '💬 ส่งใน Chat'}
                </button>
            </div>
        </div>
    );
}
```

### 8.4 Scanner Page Component

```jsx
// src/pages/Scanner.jsx
import { useState } from 'react';
import { useQRScanner } from '../hooks/useQRScanner';
import { useMessages } from '../hooks/useMessages';

export default function ScannerPage() {
    const { scanning, lastResult, history, error, scan, clearHistory, canScan } = useQRScanner();
    const { sendText, sending } = useMessages();
    const [selectedResult, setSelectedResult] = useState(null);
    
    const handleSendResult = async (value) => {
        try {
            await sendText(`🔍 ผลการสแกน QR Code\n━━━━━━━━━━━━\n${value}`);
            alert('ส่งผลการสแกนใน Chat แล้ว!');
        } catch (error) {
            alert('เกิดข้อผิดพลาด: ' + error.message);
        }
    };
    
    const handleCopy = (value) => {
        navigator.clipboard.writeText(value)
            .then(() => alert('Copy แล้ว!'))
            .catch(() => alert('ไม่สามารถ Copy ได้'));
    };
    
    return (
        <div style={{ padding: '24px 16px 100px' }}>
            <h2 style={{ fontSize: 20, fontWeight: 700, marginBottom: 24 }}>QR Scanner</h2>
            
            {/* Scanner Button */}
            <div style={{ background: 'linear-gradient(135deg, #1e293b, #334155)',
                         borderRadius: 20, padding: '32px 24px',
                         textAlign: 'center', marginBottom: 16 }}>
                <div style={{ fontSize: 56, marginBottom: 16 }}>📷</div>
                <h3 style={{ color: 'white', marginBottom: 8 }}>สแกน QR Code</h3>
                <p style={{ color: '#94a3b8', fontSize: 13, marginBottom: 24 }}>
                    {canScan ? 'กดปุ่มด้านล่างเพื่อเปิดกล้อง' : 'ต้องเปิดจาก LINE App'}
                </p>
                
                <button
                    onClick={scan}
                    disabled={scanning || !canScan}
                    style={{
                        padding: '14px 40px',
                        background: scanning || !canScan ? '#374155' : '#06c755',
                        color: 'white',
                        border: 'none',
                        borderRadius: 12,
                        fontSize: 16,
                        fontWeight: 700,
                        cursor: scanning || !canScan ? 'not-allowed' : 'pointer'
                    }}
                >
                    {scanning ? '⏳ กำลังสแกน...' : '🔍 เปิดกล้อง'}
                </button>
            </div>
            
            {/* Error */}
            {error && (
                <div style={{ background: '#fee2e2', color: '#dc2626', padding: '12px 16px',
                             borderRadius: 10, marginBottom: 16, fontSize: 14 }}>
                    ❌ {error}
                </div>
            )}
            
            {/* Last Result */}
            {lastResult && (
                <div style={{ background: '#f0fdf4', border: '1.5px solid #86efac',
                             borderRadius: 16, padding: 16, marginBottom: 16 }}>
                    <div style={{ fontSize: 12, fontWeight: 700, color: '#166534',
                                textTransform: 'uppercase', marginBottom: 8 }}>
                        ✅ ผลล่าสุด
                    </div>
                    <div style={{ fontSize: 14, color: '#14532d', wordBreak: 'break-all',
                                fontFamily: 'monospace', marginBottom: 12 }}>
                        {lastResult.value}
                    </div>
                    <div style={{ display: 'flex', gap: 8 }}>
                        <button
                            onClick={() => handleCopy(lastResult.value)}
                            style={{ flex: 1, padding: '8px', background: '#dcfce7',
                                   color: '#166534', border: 'none', borderRadius: 8,
                                   fontSize: 13, fontWeight: 600, cursor: 'pointer' }}
                        >
                            📋 Copy
                        </button>
                        <button
                            onClick={() => handleSendResult(lastResult.value)}
                            disabled={sending}
                            style={{ flex: 1, padding: '8px', background: '#06c755',
                                   color: 'white', border: 'none', borderRadius: 8,
                                   fontSize: 13, fontWeight: 600,
                                   cursor: sending ? 'not-allowed' : 'pointer' }}
                        >
                            {sending ? '...' : '💬 ส่ง'}
                        </button>
                    </div>
                </div>
            )}
            
            {/* History */}
            {history.length > 0 && (
                <div style={{ background: 'white', borderRadius: 16,
                             boxShadow: '0 2px 8px rgba(0,0,0,0.08)', overflow: 'hidden' }}>
                    <div style={{ padding: '14px 16px', background: '#f8f9fa',
                                 borderBottom: '1px solid #e5e7eb',
                                 display: 'flex', justifyContent: 'space-between',
                                 alignItems: 'center' }}>
                        <span style={{ fontSize: 13, fontWeight: 700, color: '#666',
                                     textTransform: 'uppercase' }}>
                            ประวัติการสแกน ({history.length})
                        </span>
                        <button
                            onClick={clearHistory}
                            style={{ background: 'none', border: 'none',
                                   color: '#dc2626', fontSize: 13, cursor: 'pointer' }}
                        >
                            ล้าง
                        </button>
                    </div>
                    
                    {history.map(item => (
                        <div
                            key={item.id}
                            onClick={() => setSelectedResult(selectedResult === item.id ? null : item.id)}
                            style={{ padding: '12px 16px', borderBottom: '1px solid #f3f4f6',
                                   cursor: 'pointer' }}
                        >
                            <div style={{ fontSize: 11, color: '#9ca3af', marginBottom: 4 }}>
                                {item.timestamp.toLocaleTimeString('th-TH')}
                            </div>
                            <div style={{ fontSize: 13, color: '#374151', fontFamily: 'monospace',
                                       wordBreak: 'break-all',
                                       overflow: selectedResult === item.id ? 'visible' : 'hidden',
                                       textOverflow: 'ellipsis',
                                       whiteSpace: selectedResult === item.id ? 'normal' : 'nowrap' }}>
                                {item.value}
                            </div>
                        </div>
                    ))}
                </div>
            )}
            
            {history.length === 0 && !lastResult && (
                <div style={{ textAlign: 'center', padding: '32px', color: '#9ca3af' }}>
                    <div style={{ fontSize: 48, marginBottom: 12 }}>📋</div>
                    <p>ยังไม่มีประวัติการสแกน</p>
                </div>
            )}
        </div>
    );
}
```

### 8.5 Common Components

```jsx
// src/components/common/LoadingScreen.jsx
export default function LoadingScreen({ message = 'กำลังโหลด...' }) {
    return (
        <div style={{
            display: 'flex',
            flexDirection: 'column',
            alignItems: 'center',
            justifyContent: 'center',
            minHeight: '100vh',
            gap: 16,
            background: '#f7f8fa'
        }}>
            <style>{`
                @keyframes spin { to { transform: rotate(360deg); } }
            `}</style>
            <div style={{
                width: 44,
                height: 44,
                borderRadius: '50%',
                border: '4px solid #e5e7eb',
                borderTopColor: '#06c755',
                animation: 'spin 0.7s linear infinite'
            }} />
            <p style={{ color: '#64748b', fontSize: 14, margin: 0 }}>{message}</p>
        </div>
    );
}

// src/components/common/ErrorScreen.jsx
export default function ErrorScreen({ message, onRetry }) {
    return (
        <div style={{
            display: 'flex',
            flexDirection: 'column',
            alignItems: 'center',
            justifyContent: 'center',
            minHeight: '100vh',
            gap: 16,
            padding: 32,
            textAlign: 'center',
            background: '#f7f8fa'
        }}>
            <div style={{ fontSize: 64 }}>⚠️</div>
            <h2 style={{ color: '#dc2626', fontSize: 20, fontWeight: 700, margin: 0 }}>
                เกิดข้อผิดพลาด
            </h2>
            <p style={{ color: '#64748b', fontSize: 14, margin: 0 }}>{message}</p>
            {onRetry && (
                <button
                    onClick={onRetry}
                    style={{
                        padding: '12px 24px',
                        background: '#06c755',
                        color: 'white',
                        border: 'none',
                        borderRadius: 10,
                        fontSize: 15,
                        fontWeight: 600,
                        cursor: 'pointer',
                        marginTop: 8
                    }}
                >
                    ลองใหม่
                </button>
            )}
        </div>
    );
}

// src/components/common/InitGuard.jsx
import { useLiffContext } from '../../contexts/LiffContext';
import LoadingScreen from './LoadingScreen';
import ErrorScreen from './ErrorScreen';

export default function InitGuard({ children }) {
    const { loading, error, initialized } = useLiffContext();
    
    if (loading) return <LoadingScreen message="กำลังเชื่อมต่อ LINE..." />;
    if (error) return <ErrorScreen message={error} onRetry={() => window.location.reload()} />;
    if (!initialized) return <ErrorScreen message="ไม่สามารถเริ่ม LIFF ได้" />;
    
    return children;
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
vercel

# หรือ Deploy Production
vercel --prod
```

```json
// vercel.json - Configuration
{
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
    "env": {
        "VITE_LIFF_ID": "@liff-id"
    }
}
```

```bash
# ตั้งค่า Environment Variables บน Vercel
vercel env add VITE_LIFF_ID
```

### 9.2 Deploy บน Netlify

```bash
# Build
npm run build

# ติดตั้ง Netlify CLI
npm install -g netlify-cli

# Deploy
netlify deploy --dir=dist --prod
```

```toml
# netlify.toml
[build]
  publish = "dist"
  command = "npm run build"

[build.environment]
  VITE_LIFF_ID = "1234567890-AbcdEfgh"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### 9.3 GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Deploy LIFF App

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        env:
          VITE_LIFF_ID: ${{ secrets.VITE_LIFF_ID }}
          VITE_API_BASE_URL: ${{ secrets.VITE_API_BASE_URL }}
        run: npm run build
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
          vercel-args: '--prod'
```

### 9.4 Checklist ก่อน Deploy

```
Pre-deployment Checklist:
□ ทดสอบบน LINE App จริง (iOS + Android)
□ ทดสอบ Login/Logout Flow
□ ทดสอบ sendMessages
□ ทดสอบ shareTargetPicker
□ ทดสอบ QR Scanner
□ ตรวจสอบ Responsive บนทุกขนาดหน้าจอ
□ ตรวจสอบ Loading States ทั้งหมด
□ ตรวจสอบ Error Handling
□ ตรวจสอบ Environment Variables
□ อัพเดต Endpoint URL ใน LIFF Console
□ ทดสอบ HTTPS (ต้องไม่ใช่ HTTP)
□ ทดสอบ Bot Link Feature (ถ้าใช้)
□ ตรวจสอบ LIFF Size เหมาะสม
□ Performance Test (Lighthouse Score)
```

---

## 10. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Complete Profile App

สร้าง React LIFF App ที่มี:
1. **ProfileCard Component** - แสดงรูปโปรไฟล์, ชื่อ, User ID
2. **useLiff Hook** - จัดการ LIFF State ทั้งหมด
3. **useProfile Hook** - Profile-specific operations
4. **LiffContext** - Share State ระหว่าง Component
5. **Loading/Error States** ที่ดี

```
โครงสร้าง:
App
└── LiffProvider
    └── ProfilePage
        ├── LiffInfo (OS, Language, etc.)
        ├── ProfileCard (Avatar, Name, ID)
        └── ActionButtons (Send, Logout)
```

### แบบฝึกหัดที่ 2: Message Builder App

สร้าง React LIFF App ที่:
1. เลือก Message Template จาก Dropdown
2. Fill in ตัวแปรในข้อความ (เช่น ชื่อ, วันที่)
3. Preview ข้อความก่อนส่ง
4. ส่งข้อความไปใน Chat
5. หรือแชร์ไปยัง Chat อื่น

### แบบฝึกหัดที่ 3: Mini E-commerce LIFF

สร้าง Mini E-commerce App ครบวงจร:

```
Pages:
1. Home - Banner + Featured Products
2. Products - Product Grid + Filter
3. Product Detail - LIFF opens with product ID
4. Cart - Cart Items (Zustand)
5. Checkout - Form + Send Order to Chat

Hooks ที่ต้องสร้าง:
- useLiff() - LIFF initialization
- useProfile() - User profile
- useCart() - Cart management (Zustand)
- useMessages() - Send messages
- useProducts() - Product data fetching

Features:
- Add/Remove ใน Cart
- Checkout ส่ง Order Summary Flex Message
- Share Product Card ให้เพื่อน
- QR Scanner รับ Coupon Code
```

**Starter Code:**

```jsx
// App.jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { LiffProvider } from './contexts/LiffContext';
import InitGuard from './components/common/InitGuard';
import BottomNav from './components/layout/BottomNav';

export default function App() {
    return (
        <LiffProvider liffId={import.meta.env.VITE_LIFF_ID}>
            <BrowserRouter>
                <InitGuard>
                    <div className="app">
                        <Routes>
                            {/* TODO: เพิ่ม Routes */}
                        </Routes>
                        <BottomNav />
                    </div>
                </InitGuard>
            </BrowserRouter>
        </LiffProvider>
    );
}
```

---

## สรุป

### สิ่งที่ได้เรียนรู้ใน Part 45

| หัวข้อ | เนื้อหา |
|--------|---------|
| Setup | Vite + React + TypeScript + LIFF SDK |
| LiffContext | Context Provider สำหรับ Share LIFF State |
| Custom Hooks | useLiff, useProfile, useMessages, useQRScanner |
| React Router | Routing + Deep Links + Protected Routes |
| Zustand | State Management สำหรับ Cart |
| Components | Profile, Messages, QR Scanner |
| Deployment | Vercel, Netlify, GitHub Actions |

### Architecture Pattern ที่แนะนำ

```
App
├── LiffProvider (Context)
├── BrowserRouter
│   └── AppShell (Layout)
│       ├── Pages
│       │   ├── Home
│       │   ├── Profile → useProfile()
│       │   ├── Messages → useMessages()
│       │   └── Scanner → useQRScanner()
│       └── BottomNav
└── Zustand Stores
    ├── useLiffStore (LIFF State)
    └── useCartStore (Cart)
```

### ขั้นตอนต่อไป

การเรียนรู้ LIFF กับ React เป็นพื้นฐานที่ดีสำหรับ:
- **Next.js + LIFF** - สำหรับ SEO และ SSR
- **React Native + LINE SDK** - สำหรับ Native App
- **LIFF + Payment Gateway** - ระบบชำระเงิน
- **LIFF + WebRTC** - Video/Audio Calls

---

*บทเรียนชุดนี้ครอบคลุม LIFF ตั้งแต่พื้นฐานจนถึง Production-Ready React App*
*จาก Part 41 (LIFF Introduction) ถึง Part 45 (LIFF + React)*
