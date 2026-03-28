---
name: vue3-best-practices
description: Use this skill when working on Vue 3 projects with TypeScript. It provides Chinese-language best practices for Vue 3 Composition API, Pinia state management, Element Plus UI, VueRouter, and project structure conventions commonly used in Chinese development teams.
---

# Vue 3 + TypeScript 中文开发规范

## 项目结构

```
src/
├── api/                # API 请求封装
│   ├── modules/        # 按模块拆分的 API
│   └── request.ts      # axios 实例与拦截器
├── assets/             # 静态资源
├── components/         # 公共组件
│   ├── business/       # 业务组件
│   └── common/         # 通用组件（按钮、弹窗等）
├── composables/        # 组合式函数（useXxx）
├── directives/         # 自定义指令
├── hooks/              # 通用 hooks
├── layouts/            # 布局组件
├── pages/              # 页面（或 views/）
├── router/             # 路由配置
│   ├── index.ts
│   └── modules/        # 按模块拆分的路由
├── stores/             # Pinia 状态管理
│   └── modules/        # 按模块拆分的 store
├── styles/             # 全局样式
│   ├── variables.scss  # SCSS 变量
│   └── index.scss      # 全局入口
├── types/              # TypeScript 类型定义
├── utils/              # 工具函数
├── App.vue
└── main.ts
```

## 命名规范

### 文件命名
- 组件文件：PascalCase，如 `UserProfile.vue`、`SearchBar.vue`
- 组合式函数：camelCase + use 前缀，如 `useUserList.ts`、`useAuth.ts`
- 工具函数：camelCase，如 `formatDate.ts`、`validate.ts`
- 类型文件：camelCase，如 `user.d.ts`、`api.d.ts`
- Store 文件：camelCase，如 `userStore.ts`

### 组件命名
- 组件名必须是多词的，避免和 HTML 元素冲突：`UserCard` 而非 `Card`
- 基础组件用 `Base` 前缀：`BaseButton`、`BaseInput`
- 业务组件用业务域前缀：`OrderList`、`UserAvatar`

### 变量命名
- 响应式变量：直接语义化，如 `const userList = ref<User[]>([])`
- 布尔值：is/has/can 前缀，如 `isLoading`、`hasPermission`
- 事件处理：handle 前缀，如 `handleSubmit`、`handleDelete`
- 计算属性：语义化名词，如 `filteredList`、`totalPrice`

## Composition API 规范

### setup 内部顺序
```typescript
// 1. 导入
import { ref, computed, onMounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores/modules/userStore'

// 2. Props 和 Emits 定义
const props = defineProps<{
  userId: number
  showAvatar?: boolean
}>()

const emit = defineEmits<{
  update: [value: string]
  delete: [id: number]
}>()

// 3. 响应式状态
const loading = ref(false)
const userList = ref<User[]>([])

// 4. 计算属性
const activeUsers = computed(() =>
  userList.value.filter(u => u.status === 'active')
)

// 5. 方法
async function fetchUsers() {
  loading.value = true
  try {
    userList.value = await getUserList()
  } finally {
    loading.value = false
  }
}

// 6. 侦听器
watch(() => props.userId, (newId) => {
  if (newId) fetchUsers()
})

// 7. 生命周期
onMounted(() => {
  fetchUsers()
})
```

### 组合式函数（Composables）
```typescript
// composables/useUserList.ts
export function useUserList() {
  const users = ref<User[]>([])
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function fetch(params?: UserQueryParams) {
    loading.value = true
    error.value = null
    try {
      users.value = await api.user.getList(params)
    } catch (e) {
      error.value = (e as Error).message
    } finally {
      loading.value = false
    }
  }

  return { users, loading, error, fetch }
}
```

## Pinia 状态管理

### Store 定义
```typescript
// stores/modules/userStore.ts
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', () => {
  // 状态
  const token = ref('')
  const userInfo = ref<UserInfo | null>(null)

  // 计算属性
  const isLoggedIn = computed(() => !!token.value)
  const userName = computed(() => userInfo.value?.name ?? '未登录')

  // 操作
  async function login(credentials: LoginParams) {
    const res = await api.auth.login(credentials)
    token.value = res.token
    userInfo.value = res.userInfo
  }

  function logout() {
    token.value = ''
    userInfo.value = null
  }

  return { token, userInfo, isLoggedIn, userName, login, logout }
}, {
  persist: true, // pinia-plugin-persistedstate
})
```

### Store 使用原则
- 只在 store 中存放跨组件共享的状态
- 组件内部状态用 `ref`/`reactive`，不要放 store
- Store 之间避免循环依赖
- 异步操作放在 store 的 action 中

## API 请求封装

```typescript
// api/request.ts
import axios from 'axios'
import { useUserStore } from '@/stores/modules/userStore'
import { ElMessage } from 'element-plus'

const request = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15000,
})

// 请求拦截器
request.interceptors.request.use((config) => {
  const userStore = useUserStore()
  if (userStore.token) {
    config.headers.Authorization = `Bearer ${userStore.token}`
  }
  return config
})

// 响应拦截器
request.interceptors.response.use(
  (response) => response.data,
  (error) => {
    const status = error.response?.status
    const messages: Record<number, string> = {
      401: '登录已过期，请重新登录',
      403: '没有权限访问',
      404: '请求的资源不存在',
      500: '服务器内部错误',
    }
    ElMessage.error(messages[status] ?? '网络请求失败，请稍后重试')

    if (status === 401) {
      const userStore = useUserStore()
      userStore.logout()
      window.location.href = '/login'
    }

    return Promise.reject(error)
  }
)

export default request
```

## Element Plus 使用规范

- 按需导入，不要全量引入
- 表单验证用 `el-form` 的 `rules` 属性，规则集中定义
- 表格分页用 `el-pagination`，统一分页参数命名：`page`、`pageSize`
- 弹窗用 `el-dialog`，通过 `v-model` 控制显隐
- 消息提示统一用 `ElMessage`，确认框用 `ElMessageBox.confirm`

## TypeScript 规范

- 所有组件 props 必须有类型定义
- API 返回值必须有类型定义
- 避免使用 `any`，用 `unknown` + 类型守卫
- 接口定义放在 `types/` 目录，按模块拆分
- 枚举用 `const enum` 或字面量联合类型

```typescript
// types/user.d.ts
export interface User {
  id: number
  name: string
  email: string
  status: 'active' | 'inactive' | 'banned'
  createdAt: string
}

export interface UserQueryParams {
  page: number
  pageSize: number
  keyword?: string
  status?: User['status']
}

export interface UserListResponse {
  list: User[]
  total: number
}
```

## 中文注释规范

- 组件顶部加功能说明注释
- 复杂业务逻辑加注释说明"为什么"，而不是"做了什么"
- API 接口加 JSDoc 注释
- 不要给显而易见的代码加注释

```typescript
/**
 * 用户列表页
 * 支持关键词搜索、状态筛选、分页
 */

// 延迟 300ms 搜索，避免输入时频繁请求
const debouncedSearch = useDebounceFn(fetchUsers, 300)

// 删除前需要二次确认，防止误操作
async function handleDelete(id: number) {
  await ElMessageBox.confirm('确定要删除该用户吗？', '提示')
  await api.user.delete(id)
  ElMessage.success('删除成功')
  fetchUsers()
}
```
