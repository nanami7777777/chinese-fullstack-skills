---
name: wechat-miniprogram
description: Use this skill when developing WeChat Mini Programs. It provides Chinese-language best practices for WeChat Mini Program development including project structure, component design, API calls, state management, performance optimization, and common patterns for login, payment, and sharing features.
---

# 微信小程序开发规范

## 项目结构

```
├── pages/                  # 页面
│   ├── index/              # 首页
│   │   ├── index.wxml
│   │   ├── index.wxss
│   │   ├── index.ts
│   │   └── index.json
│   ├── user/               # 用户中心
│   └── order/              # 订单
├── components/             # 自定义组件
│   ├── nav-bar/            # 导航栏
│   ├── tab-bar/            # 底部导航
│   └── goods-card/         # 商品卡片
├── services/               # API 请求封装
│   ├── request.ts          # 请求基础封装
│   └── modules/            # 按模块拆分
│       ├── user.ts
│       └── order.ts
├── utils/                  # 工具函数
│   ├── auth.ts             # 登录授权
│   ├── storage.ts          # 本地存储
│   └── format.ts           # 格式化
├── store/                  # 状态管理（MobX-miniprogram）
├── styles/                 # 公共样式
│   ├── variables.wxss      # 样式变量
│   └── mixins.wxss         # 混入样式
├── typings/                # 类型定义
├── app.ts                  # 应用入口
├── app.json                # 全局配置
├── app.wxss                # 全局样式
├── project.config.json     # 项目配置
└── sitemap.json            # 搜索配置
```

## 如果使用 uni-app / Taro

### uni-app（推荐跨端方案）
```
src/
├── pages/
│   └── index/index.vue
├── components/
├── composables/            # 组合式函数
├── stores/                 # Pinia
├── utils/
├── App.vue
├── main.ts
├── manifest.json           # uni-app 配置
├── pages.json              # 页面路由
└── uni.scss                # 全局样式变量
```

### Taro（React 技术栈）
```
src/
├── pages/
│   └── index/index.tsx
├── components/
├── hooks/
├── stores/                 # Zustand
├── utils/
├── app.ts
└── app.config.ts           # Taro 配置
```

## 请求封装

```typescript
// services/request.ts
const BASE_URL = 'https://api.example.com'

interface RequestOptions {
  url: string
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE'
  data?: Record<string, unknown>
  header?: Record<string, string>
  needAuth?: boolean
}

interface ApiResponse<T = unknown> {
  code: number
  data: T
  message: string
}

export function request<T>(options: RequestOptions): Promise<T> {
  return new Promise((resolve, reject) => {
    const header: Record<string, string> = {
      'Content-Type': 'application/json',
      ...options.header,
    }

    // 需要登录的接口自动带 token
    if (options.needAuth !== false) {
      const token = wx.getStorageSync('token')
      if (token) {
        header.Authorization = `Bearer ${token}`
      }
    }

    wx.request({
      url: `${BASE_URL}${options.url}`,
      method: options.method ?? 'GET',
      data: options.data,
      header,
      success(res) {
        const data = res.data as ApiResponse<T>
        if (data.code === 0) {
          resolve(data.data)
        } else if (data.code === 401) {
          // token 过期，重新登录
          wx.removeStorageSync('token')
          wx.navigateTo({ url: '/pages/login/login' })
          reject(new Error('登录已过期'))
        } else {
          wx.showToast({ title: data.message, icon: 'none' })
          reject(new Error(data.message))
        }
      },
      fail(err) {
        wx.showToast({ title: '网络请求失败', icon: 'none' })
        reject(err)
      },
    })
  })
}
```

## 微信登录流程

```typescript
// utils/auth.ts

/** 微信登录完整流程 */
export async function wxLogin(): Promise<string> {
  // 1. 调用 wx.login 获取 code
  const { code } = await new Promise<WechatMiniprogram.LoginSuccessCallbackResult>(
    (resolve, reject) => {
      wx.login({
        success: resolve,
        fail: reject,
      })
    }
  )

  // 2. 发送 code 到后端换取 token
  const { token } = await request<{ token: string }>({
    url: '/auth/wechat-login',
    method: 'POST',
    data: { code },
    needAuth: false,
  })

  // 3. 保存 token
  wx.setStorageSync('token', token)
  return token
}

/** 检查登录状态，过期则重新登录 */
export async function checkLogin(): Promise<boolean> {
  const token = wx.getStorageSync('token')
  if (!token) {
    await wxLogin()
    return true
  }

  try {
    await new Promise<void>((resolve, reject) => {
      wx.checkSession({ success: () => resolve(), fail: () => reject() })
    })
    return true
  } catch {
    // session 过期，重新登录
    await wxLogin()
    return true
  }
}
```

## 微信支付

```typescript
// services/modules/pay.ts

/** 发起微信支付 */
export async function wxPay(orderId: string): Promise<boolean> {
  // 1. 后端创建预支付订单
  const payParams = await request<WechatMiniprogram.RequestPaymentOption>({
    url: '/pay/create',
    method: 'POST',
    data: { orderId },
  })

  // 2. 调起微信支付
  return new Promise((resolve) => {
    wx.requestPayment({
      ...payParams,
      success() {
        wx.showToast({ title: '支付成功', icon: 'success' })
        resolve(true)
      },
      fail(err) {
        if (err.errMsg.includes('cancel')) {
          wx.showToast({ title: '已取消支付', icon: 'none' })
        } else {
          wx.showToast({ title: '支付失败', icon: 'none' })
        }
        resolve(false)
      },
    })
  })
}
```

## 分享配置

```typescript
// 页面中配置分享
Page({
  /** 转发给好友 */
  onShareAppMessage() {
    return {
      title: '推荐一个好用的小程序',
      path: `/pages/index/index?shareFrom=${this.data.userId}`,
      imageUrl: '/images/share-cover.png',
    }
  },

  /** 分享到朋友圈 */
  onShareTimeline() {
    return {
      title: '推荐一个好用的小程序',
      query: `shareFrom=${this.data.userId}`,
      imageUrl: '/images/share-cover.png',
    }
  },
})
```

## 性能优化

### 启动性能
- 分包加载：主包 < 2MB，总包 < 20MB
- 按需注入：`"lazyCodeLoading": "requiredComponents"`
- 预加载分包：`preloadRule` 配置

```json
// app.json
{
  "pages": ["pages/index/index"],
  "subpackages": [
    {
      "root": "pages/order",
      "pages": ["list", "detail"]
    }
  ],
  "preloadRule": {
    "pages/index/index": {
      "network": "all",
      "packages": ["pages/order"]
    }
  },
  "lazyCodeLoading": "requiredComponents"
}
```

### 渲染性能
- 长列表用 `recycle-view`（虚拟列表）
- 图片用 `lazy-load` 懒加载
- 避免 `setData` 传递大数据，只更新变化的部分
- 减少 WXML 节点数，单页面 < 1000 个节点

### 网络性能
- 接口合并：一个页面尽量一次请求拿到所有数据
- 缓存策略：不常变的数据用 `wx.setStorageSync` 缓存
- 图片压缩：用 OSS 图片处理参数 `?x-oss-process=image/resize,w_300`

## 常见坑

- `wx.navigateTo` 最多 10 层，超过用 `wx.redirectTo`
- `setData` 是异步的，不要在 `setData` 后立即读取 `this.data`
- 小程序没有 DOM，不能用 `document.querySelector`
- `rpx` 是响应式单位，750rpx = 屏幕宽度
- 真机调试和开发者工具表现可能不同，以真机为准
- iOS 和 Android 的 `Date` 解析不同，用 `2024-01-01` 而非 `2024/01/01`
