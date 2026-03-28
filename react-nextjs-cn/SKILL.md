---
name: react-nextjs-cn
description: Use this skill when working on React or Next.js projects. It provides Chinese-language best practices for React 19, Next.js 15 App Router, Server Components, Zustand/Jotai state management, Ant Design/shadcn-ui, and project conventions for Chinese development teams.
---

# React 19 + Next.js 15 中文开发规范

## 项目结构（Next.js App Router）

```
src/
├── app/                    # App Router 页面
│   ├── (auth)/             # 路由分组：需要登录的页面
│   │   ├── dashboard/
│   │   └── settings/
│   ├── (public)/           # 路由分组：公开页面
│   │   ├── about/
│   │   └── pricing/
│   ├── api/                # API Routes
│   ├── layout.tsx          # 根布局
│   ├── page.tsx            # 首页
│   ├── loading.tsx         # 全局 loading
│   ├── error.tsx           # 全局错误边界
│   └── not-found.tsx       # 404 页面
├── components/
│   ├── ui/                 # 基础 UI 组件
│   ├── business/           # 业务组件
│   └── layouts/            # 布局组件
├── hooks/                  # 自定义 hooks
├── lib/                    # 工具库
│   ├── api.ts              # API 客户端
│   ├── auth.ts             # 认证逻辑
│   └── utils.ts            # 通用工具
├── stores/                 # 状态管理（Zustand）
├── types/                  # TypeScript 类型
└── styles/                 # 全局样式
```

## 命名规范

### 文件命名
- 组件文件：PascalCase，如 `UserCard.tsx`
- hooks：camelCase + use 前缀，如 `useAuth.ts`
- 工具函数：camelCase，如 `formatPrice.ts`
- 类型文件：camelCase，如 `user.ts`（在 types/ 下）
- API 路由：kebab-case 目录，如 `app/api/user-list/route.ts`

### 组件命名
- 服务端组件：默认，不加后缀
- 客户端组件：文件顶部加 `'use client'`
- 页面组件：`page.tsx`（Next.js 约定）
- 布局组件：`layout.tsx`（Next.js 约定）

## Server Components vs Client Components

### 默认用 Server Component
```tsx
// app/users/page.tsx — 服务端组件（默认）
import { getUsers } from '@/lib/api'
import { UserTable } from '@/components/business/UserTable'

export default async function UsersPage() {
  const users = await getUsers()
  return <UserTable users={users} />
}
```

### 需要交互时用 Client Component
```tsx
'use client'

// components/business/UserTable.tsx — 客户端组件
import { useState } from 'react'
import { User } from '@/types/user'

interface Props {
  users: User[]
}

export function UserTable({ users }: Props) {
  const [search, setSearch] = useState('')
  const filtered = users.filter(u =>
    u.name.includes(search)
  )

  return (
    <div>
      <input
        value={search}
        onChange={e => setSearch(e.target.value)}
        placeholder="搜索用户..."
      />
      <table>
        {filtered.map(user => (
          <tr key={user.id}>
            <td>{user.name}</td>
            <td>{user.email}</td>
          </tr>
        ))}
      </table>
    </div>
  )
}
```

### 原则
- 能用 Server Component 就用 Server Component
- 只在需要 useState/useEffect/事件处理/浏览器 API 时用 Client Component
- Client Component 尽量下沉到叶子节点，不要在高层组件加 `'use client'`
- 数据获取放在 Server Component，通过 props 传给 Client Component

## 状态管理（Zustand）

```typescript
// stores/userStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface UserState {
  token: string
  userInfo: UserInfo | null
  isLoggedIn: boolean
  login: (credentials: LoginParams) => Promise<void>
  logout: () => void
}

export const useUserStore = create<UserState>()(
  persist(
    (set) => ({
      token: '',
      userInfo: null,
      get isLoggedIn() { return !!this.token },

      async login(credentials) {
        const res = await fetch('/api/auth/login', {
          method: 'POST',
          body: JSON.stringify(credentials),
        }).then(r => r.json())

        set({ token: res.token, userInfo: res.user })
      },

      logout() {
        set({ token: '', userInfo: null })
      },
    }),
    { name: 'user-store' }
  )
)
```

## API 请求

### Server-side（Server Component / Server Action）
```typescript
// lib/api.ts
const API_BASE = process.env.API_BASE_URL!

export async function getUsers(): Promise<User[]> {
  const res = await fetch(`${API_BASE}/users`, {
    next: { revalidate: 60 }, // ISR：60 秒重新验证
  })
  if (!res.ok) throw new Error('获取用户列表失败')
  return res.json()
}
```

### Client-side（Client Component）
```typescript
// hooks/useUsers.ts
'use client'
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(r => r.json())

export function useUsers(params?: { page: number }) {
  const searchParams = new URLSearchParams()
  if (params?.page) searchParams.set('page', String(params.page))

  return useSWR<UserListResponse>(
    `/api/users?${searchParams}`,
    fetcher
  )
}
```

## Server Actions

```typescript
// app/users/actions.ts
'use server'

import { revalidatePath } from 'next/cache'

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string
  const email = formData.get('email') as string

  // 服务端验证
  if (!name || !email) {
    return { error: '姓名和邮箱不能为空' }
  }

  await db.user.create({ data: { name, email } })
  revalidatePath('/users')
  return { success: true }
}
```

## TypeScript 规范

```typescript
// types/user.ts
export interface User {
  id: string
  name: string
  email: string
  avatar?: string
  role: 'admin' | 'editor' | 'viewer'
  createdAt: string
}

export interface LoginParams {
  email: string
  password: string
}

export interface UserListResponse {
  users: User[]
  total: number
  page: number
  pageSize: number
}
```

## 中文注释规范

- 页面组件顶部加功能说明
- 复杂业务逻辑说明"为什么"
- Server Action 加参数和返回值说明
- 不要给 JSX 结构加注释，结构本身就是文档

```tsx
/**
 * 用户管理页面
 * - 支持分页、搜索、角色筛选
 * - 管理员可以编辑和删除用户
 */
export default async function UsersPage({ searchParams }: Props) {
  // searchParams 在 Next.js 15 中是 Promise，需要 await
  const params = await searchParams
  const users = await getUsers(params)
  return <UserTable users={users} />
}
```

## 常用库推荐

| 用途 | 推荐 | 备注 |
|------|------|------|
| UI 组件 | Ant Design 5 / shadcn-ui | 国内项目推荐 Ant Design |
| 状态管理 | Zustand | 轻量，替代 Redux |
| 数据请求 | SWR / TanStack Query | Client-side 数据获取 |
| 表单 | React Hook Form + Zod | 类型安全的表单验证 |
| 日期 | dayjs | 替代 moment.js |
| 图表 | ECharts / Recharts | 国内项目推荐 ECharts |
| 国际化 | next-intl | Next.js 国际化 |
