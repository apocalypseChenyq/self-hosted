# Vue3项目接入Sentry完整指南（适用于Sentry 9.x）

## 目录
- [1. 环境准备](#1-环境准备)
- [2. Sentry后台配置](#2-sentry后台配置)
- [3. Vue3项目接入](#3-vue3项目接入)
- [4. 高级配置](#4-高级配置)
- [5. 错误监控最佳实践](#5-错误监控最佳实践)
- [6. Session Replay功能](#6-session-replay功能)
- [7. 性能监控配置](#7-性能监控配置)
- [8. Sentry后台使用说明](#8-sentry后台使用说明)
- [9. 参数详解](#9-参数详解sentry-9x版本)
- [10. 常见问题与解决方案](#10-常见问题与解决方案)
- [11. 版本升级指导](#11-版本升级指导)
- [12. 最佳实践总结](#12-最佳实践总结)
- [13. 版本管理和更新策略](#13-版本管理和更新策略)
- [14. 性能优化指南](#14-性能优化指南)
- [15. 成本控制策略](#15-成本控制策略)
- [16. 应急处理和容错](#16-应急处理和容错)

## 1. 环境准备

### 前置条件
- ✅ Sentry自托管服务已部署并运行
- ✅ Vue3项目已创建
- ✅ Node.js >= 18.0.0
- ✅ npm/yarn包管理器

### 服务地址确认
- Sentry服务地址：`http://localhost:9000`
- API地址：`http://localhost:9000/api/`

## 2. Sentry后台配置

### 2.1 创建管理员账户

首先需要创建Sentry管理员账户：

```bash
# 在Sentry安装目录执行
docker-compose exec web sentry createuser
```

按提示输入：
- Email地址（用作登录账号）
- 密码
- 是否为超级用户（选择 y）

### 2.2 登录Sentry后台

1. 访问 `http://localhost:9000`
2. 使用刚创建的账户登录
3. 首次登录会进入设置向导

### 2.3 创建组织和项目

#### 创建组织
1. 登录后点击 **"Create Organization"**
2. 填写组织信息：
   - **Organization Name**: `Your Company`
   - **Short Name**: `your-company`
3. 点击 **"Create Organization"**

#### 创建项目
1. 在组织页面点击 **"Create Project"**
2. 选择平台：**"Vue"**
3. 填写项目信息：
   - **Project Name**: `vue3-frontend`
   - **Team**: 选择默认团队
4. 点击 **"Create Project"**

### 2.4 获取DSN

项目创建成功后，系统会显示DSN（Data Source Name）：
```
http://[PUBLIC_KEY]@localhost:9000/[PROJECT_ID]
```

**重要**：请保存好这个DSN，后续配置需要使用。

## 3. Vue3项目接入

### 3.1 安装Sentry SDK

**重要**：从Sentry 9.x版本开始，不再需要单独安装`@sentry/tracing`包，所有功能已整合到主包中。

```bash
# 使用npm
npm install @sentry/vue --save

# 使用yarn
yarn add @sentry/vue

# 使用pnpm
pnpm add @sentry/vue
```

**版本要求**：
- Sentry SDK: 9.x及以上版本（当前最新版本：9.32.0）
- Vue: 3.x版本
- Node.js: 18.0.0及以上（Sentry 9.x要求）
- TypeScript: 5.0.4及以上（如果使用TypeScript）
- 浏览器兼容性：
  - Chrome 80+ (2020年2月)
  - Edge 80+ (2020年2月)
  - Safari 14+ (2020年9月)
  - Firefox 74+ (2020年3月)
  - 不支持IE11（需要ES2020支持）

### 3.2 重要变更说明（Sentry 9.x）

**🚨 重要变更提醒**：

1. **IP地址收集变更**：
   - Sentry 9.x默认不再自动收集用户IP地址
   - 如需IP地址信息，必须显式设置 `sendDefaultPii: true`
   - 影响用户识别和地理位置统计

2. **集成语法变更**：
   - 所有集成从 `new Integration()` 改为 `Sentry.integration()` 函数调用
   - 移除了 `@sentry/tracing` 包依赖

3. **TypeScript支持**：
   - 最低要求TypeScript 5.0.4
   - 内置类型定义，无需额外安装 `@types/sentry`

### 3.3 基础配置

#### 创建Sentry配置文件

创建 `src/utils/sentry.js`：

```javascript
import { createApp } from 'vue'
import * as Sentry from '@sentry/vue'

export function setupSentry(app, router) {
  Sentry.init({
    app,
    // DSN - 从Sentry后台获取
    dsn: 'http://[YOUR_DSN]@localhost:9000/[PROJECT_ID]',

    // 环境配置
    environment: process.env.NODE_ENV || 'development',

    // 发布版本
    release: process.env.VUE_APP_VERSION || '1.0.0',

    // 采样率配置
    tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,

    // Session Replay采样率（Sentry 9.x新特性）
    replaysSessionSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
    replaysOnErrorSampleRate: 1.0, // 发生错误时100%录制

    // 集成配置（Sentry 9.x新语法）
    integrations: [
      // 浏览器性能追踪
      Sentry.browserTracingIntegration({
        router, // 传入Vue Router实例
        // 路由追踪配置
        trackComponents: true,
        // 传播目标（重要：控制分布式追踪范围）
        tracePropagationTargets: ['localhost', 'your-api-domain.com', /^\//],
      }),

      // Session Replay集成（新功能）
      Sentry.replayIntegration({
        // 隐私保护配置
        maskAllText: true, // 隐藏所有文本内容
        blockAllMedia: true, // 阻止所有媒体内容
        maskAllInputs: true, // 隐藏所有输入框内容

        // 网络请求配置
        networkDetailAllowUrls: [], // 允许记录详细信息的URL
        networkCaptureBodies: false, // 不捕获请求体
        networkRequestHeaders: [], // 记录的请求头
        networkResponseHeaders: [], // 记录的响应头
      }),

      // 用户反馈集成
      Sentry.feedbackIntegration({
        colorScheme: 'system',
        showBranding: false,
      }),

      // Canvas录制集成（可选，用于录制Canvas元素）
      // Sentry.replayCanvasIntegration(),
    ],

    // 错误过滤
    beforeSend(event, hint) {
      // 开发环境下在控制台也打印错误
      if (process.env.NODE_ENV === 'development') {
        console.error('Sentry Error:', event, hint)
      }

      // 过滤掉某些错误
      if (event.exception) {
        const error = hint.originalException
        // 忽略网络错误
        if (error && error.name === 'NetworkError') {
          return null
        }
        // 忽略脚本加载错误
        if (event.exception.values[0]?.value?.includes('Script error')) {
          return null
        }
      }

      return event
    },

    // 用户信息
    initialScope: {
      tags: {
        component: 'vue3-frontend'
      }
    },

    // 调试模式
    debug: process.env.NODE_ENV === 'development',

    // 🚨 重要：IP地址和用户信息收集（Sentry 9.x变更）
    // 设为true以收集IP地址和用户代理信息，便于用户识别和地理统计
    // 生产环境根据隐私政策决定是否启用
    sendDefaultPii: true, // 建议开启以获得完整的用户上下文
  })
}
```

#### 在main.js中初始化

```javascript
// main.js
import { createApp } from 'vue'
import { createRouter, createWebHistory } from 'vue-router'
import App from './App.vue'
import { setupSentry } from './utils/sentry'

const app = createApp(App)

// 创建路由实例
const router = createRouter({
  history: createWebHistory(),
  routes: [
    // 你的路由配置
  ]
})

// 配置Sentry（注意：在app.use(router)之前，但router必须已创建）
setupSentry(app, router)

app.use(router)
app.mount('#app')
```

#### 环境变量配置

创建 `.env` 文件：

```bash
# .env.development
VUE_APP_SENTRY_DSN=http://[YOUR_DSN]@localhost:9000/[PROJECT_ID]
VUE_APP_SENTRY_ENVIRONMENT=development
VUE_APP_VERSION=1.0.0-dev

# .env.production
VUE_APP_SENTRY_DSN=http://[YOUR_DSN]@localhost:9000/[PROJECT_ID]
VUE_APP_SENTRY_ENVIRONMENT=production
VUE_APP_VERSION=1.0.0
```

更新Sentry配置以使用环境变量：

```javascript
// src/utils/sentry.js
export function setupSentry(app, router) {
  Sentry.init({
    app,
    dsn: process.env.VUE_APP_SENTRY_DSN,
    environment: process.env.VUE_APP_SENTRY_ENVIRONMENT,
    release: process.env.VUE_APP_VERSION,
    // ... 其他配置
  })
}
```

## 4. 高级配置

### 4.1 用户上下文设置

```javascript
// 在用户登录后设置用户信息
import * as Sentry from '@sentry/vue'

// 用户登录成功后
function onUserLogin(user) {
  Sentry.setUser({
    id: user.id,
    email: user.email,
    username: user.username,
    // 其他用户信息
  })

  Sentry.setTag('user_type', user.type)
  Sentry.setTag('user_role', user.role)
}

// 用户登出时清除信息
function onUserLogout() {
  Sentry.setUser(null)
}
```

### 4.2 自定义错误上报

```javascript
// utils/errorHandler.js
import * as Sentry from '@sentry/vue'

// 手动捕获异常
export function captureError(error, context = {}) {
  Sentry.withScope((scope) => {
    // 设置额外上下文
    scope.setTag('error_type', 'manual')
    scope.setContext('additional_info', context)

    // 上报错误
    Sentry.captureException(error)
  })
}

// 记录面包屑
export function addBreadcrumb(message, category = 'custom', level = 'info') {
  Sentry.addBreadcrumb({
    message,
    category,
    level,
    timestamp: Date.now() / 1000
  })
}

// API请求错误处理
export function handleApiError(error, url, method) {
  Sentry.withScope((scope) => {
    scope.setTag('error_type', 'api')
    scope.setContext('request_info', {
      url,
      method,
      status: error.response?.status,
      statusText: error.response?.statusText
    })

    Sentry.captureException(error)
  })
}
```

### 4.3 性能监控

```javascript
// utils/performance.js
import * as Sentry from '@sentry/vue'

// 自定义性能监控（Sentry 9.x新API）
export function measurePerformance(name, fn) {
  return Sentry.startSpan({ name, op: 'custom' }, (span) => {
    try {
      const result = fn()
      span.setStatus('ok')
      return result
    } catch (error) {
      span.setStatus('internal_error')
      throw error
    }
  })
}

// 异步操作性能监控
export function measureAsyncPerformance(name, asyncFn) {
  return Sentry.startSpan({ name, op: 'custom' }, async (span) => {
    try {
      const result = await asyncFn()
      span.setStatus('ok')
      return result
    } catch (error) {
      span.setStatus('internal_error')
      throw error
    }
  })
}

// 页面加载性能监控
export function trackPageLoad(routeName) {
  // Sentry 9.x 会自动进行页面加载性能监控
  // 手动添加面包屑用于调试
  Sentry.addBreadcrumb({
    message: `页面加载: ${routeName}`,
    category: 'navigation',
    level: 'info'
  })
}
```

### 4.4 Vue组件错误边界

创建错误边界组件：

```vue
<!-- components/ErrorBoundary.vue -->
<template>
  <div v-if="hasError" class="error-boundary">
    <h2>页面出现错误</h2>
    <p>{{ errorMessage }}</p>
    <button @click="retry">重试</button>
  </div>
  <slot v-else />
</template>

<script>
import * as Sentry from '@sentry/vue'

export default {
  name: 'ErrorBoundary',
  data() {
    return {
      hasError: false,
      errorMessage: ''
    }
  },
  errorCaptured(error, instance, info) {
    this.hasError = true
    this.errorMessage = error.message

    // 上报到Sentry（Sentry 9.x API）
    Sentry.withScope((scope) => {
      scope.setTag('error_boundary', true)
      scope.setContext('vue_info', {
        componentName: instance.$options?.name || 'UnknownComponent',
        info
      })

      // 添加额外的调试信息
      scope.setContext('error_boundary', {
        hasError: this.hasError,
        componentStack: info
      })

      Sentry.captureException(error)
    })

    return false // 阻止错误继续向上传播
  },
  methods: {
    retry() {
      this.hasError = false
      this.errorMessage = ''
      this.$emit('retry') // 通知父组件重试
    }
  }
}
</script>

<style scoped>
.error-boundary {
  padding: 20px;
  border: 1px solid #ff6b6b;
  border-radius: 8px;
  background-color: #fff5f5;
  color: #c92a2a;
  text-align: center;
}

.error-boundary button {
  margin-top: 10px;
  padding: 8px 16px;
  background-color: #ff6b6b;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.error-boundary button:hover {
  background-color: #fa5252;
}
</style>
```

### 4.5 生产环境Source Map配置（强烈推荐）

#### 为什么生产环境必须配置Source Map

在生产环境中，JavaScript代码会经过压缩和混淆，没有Source Map的错误信息是这样的：

**❌ 没有Source Map - 无法调试**：
```
TypeError: Cannot read property 'map' of undefined
    at n (main.js:1:2847)
    at r (main.js:1:3251)
    at Object.handleClick (main.js:1:4789)
```

**✅ 有Source Map - 精确定位**：
```
TypeError: Cannot read property 'map' of undefined
    at UserList.vue:45:12 (renderUserList)
    at UserList.vue:23:8 (fetchUsers)
    at LoginPage.vue:67:15 (handleLogin)
```

#### 配置步骤

**1. 安装Sentry CLI**

```bash
# 全局安装（推荐）
npm install -g @sentry/cli

# 或在项目中安装
npm install --save-dev @sentry/cli
```

**2. 配置身份验证**

创建 `.sentryclirc` 文件：

```ini
[defaults]
url = http://localhost:9000
org = your-organization-name
project = vue3-frontend

[auth]
token = YOUR_AUTH_TOKEN
```

**3. 获取Auth Token**

在Sentry后台：
1. 点击右上角头像 → Settings
2. 选择 Account → API
3. 点击 "Create New Token"
4. 选择权限：`project:read`, `project:write`, `project:releases`
5. 复制生成的token

**4. 自动配置Source Map（推荐使用Sentry Wizard）**

**使用Sentry Wizard自动配置**（官方推荐方式）：
```bash
# 运行Sentry Wizard，自动配置Source Maps
npx @sentry/wizard@latest -i sourcemaps
```

Wizard会自动完成以下配置：
- 登录Sentry并选择项目
- 安装必要的Sentry包
- 配置构建工具生成和上传Source Maps
- 配置CI自动上传

**手动配置Vite项目**（如果不使用Wizard）：
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  build: {
    // 生产环境生成source map
    sourcemap: true,
    rollupOptions: {
      output: {
        // 规范化文件名，便于Sentry识别
        entryFileNames: 'js/[name]-[hash].js',
        chunkFileNames: 'js/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash].[ext]'
      }
    }
  }
})
```

**Vue CLI/Webpack项目配置**：
```javascript
// vue.config.js
module.exports = {
  // 生产环境生成source map
  productionSourceMap: true,
  configureWebpack: {
    devtool: 'source-map'
  }
}
```

**5. 自动化上传脚本**

更新 `package.json`：

```json
{
  "scripts": {
    "build": "vite build",
    "build:production": "npm run build && npm run sentry:release",
    "sentry:release": "sentry-cli releases new $npm_package_version && sentry-cli releases files $npm_package_version upload-sourcemaps ./dist --rewrite && sentry-cli releases finalize $npm_package_version && npm run clean:sourcemaps",
    "clean:sourcemaps": "find ./dist -name '*.map' -delete"
  }
}
```

**6. 环境变量配置**

```bash
# .env.production
VITE_APP_SENTRY_DSN=http://[YOUR_DSN]@localhost:9000/[PROJECT_ID]
VITE_APP_VERSION=1.0.0
SENTRY_ORG=your-organization-name
SENTRY_PROJECT=vue3-frontend
SENTRY_AUTH_TOKEN=your-auth-token
```

**7. CI/CD集成示例**

```yaml
# .github/workflows/deploy.yml
name: Deploy Production
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

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build
        env:
          VITE_APP_VERSION: ${{ github.sha }}

      - name: Create Sentry Release
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: your-organization
          SENTRY_PROJECT: vue3-frontend
        run: |
          # 创建发布版本
          sentry-cli releases new ${{ github.sha }}

          # 上传Source Maps
          sentry-cli releases files ${{ github.sha }} upload-sourcemaps ./dist --rewrite

          # 关联Git提交信息
          sentry-cli releases set-commits ${{ github.sha }} --auto

          # 完成发布
          sentry-cli releases finalize ${{ github.sha }}

          # 清理.map文件（安全考虑）
          find ./dist -name "*.map" -delete
```

#### 安全最佳实践

1. **上传后删除Source Map文件**：
```bash
# 确保.map文件不会部署到生产服务器
find ./dist -name "*.map" -delete
```

2. **限制Source Map访问权限**：
- 只有开发团队可以查看Sentry中的源码
- 设置适当的团队权限

3. **使用专用的CI Token**：
- 为CI/CD创建专门的token
- 定期轮换token

#### 验证Source Map是否生效

**方法1：在代码中添加测试按钮**（推荐）
```vue
<!-- App.vue -->
<template>
  <div>
    <!-- 其他内容 -->
    <button @click="throwError">Test Sentry Error</button>
  </div>
</template>

<script>
export default {
  methods: {
    throwError() {
      throw new Error('Sentry Error - Source Map Test');
    }
  }
}
</script>
```

**方法2：在浏览器控制台测试**
```javascript
// 在生产环境控制台执行
throw new Error('测试Source Map是否正常工作')
```

在Sentry后台查看错误时，应该看到：
- ✅ 准确的文件名（如：`LoginPage.vue:45:12`）
- ✅ 可读的源码行号
- ✅ 完整的组件调用栈

## 5. 错误监控最佳实践

### 5.1 全局错误处理

```javascript
// main.js
import { captureError } from './utils/errorHandler'

// Vue全局错误处理
app.config.errorHandler = (error, instance, info) => {
  console.error('Vue Error:', error, info)
  captureError(error, {
    vue_info: info,
    component_name: instance?.$options.name
  })
}

// 全局未捕获错误
window.addEventListener('error', (event) => {
  captureError(new Error(event.message), {
    filename: event.filename,
    lineno: event.lineno,
    colno: event.colno
  })
})

// Promise未捕获错误
window.addEventListener('unhandledrejection', (event) => {
  captureError(event.reason, {
    type: 'unhandled_promise_rejection'
  })
})
```

### 5.2 API请求监控

```javascript
// api/request.js
import axios from 'axios'
import { handleApiError, addBreadcrumb } from '@/utils/errorHandler'

const api = axios.create({
  baseURL: process.env.VUE_APP_API_BASE_URL
})

// 请求拦截器
api.interceptors.request.use(
  (config) => {
    addBreadcrumb(
      `API Request: ${config.method?.toUpperCase()} ${config.url}`,
      'http',
      'info'
    )
    return config
  },
  (error) => {
    handleApiError(error, 'unknown', 'unknown')
    return Promise.reject(error)
  }
)

// 响应拦截器
api.interceptors.response.use(
  (response) => {
    addBreadcrumb(
      `API Response: ${response.status} ${response.config.url}`,
      'http',
      'info'
    )
    return response
  },
  (error) => {
    const config = error.config || {}
    handleApiError(error, config.url, config.method)
    return Promise.reject(error)
  }
)

export default api
```

### 5.3 关键业务流程监控

```javascript
// 在关键业务操作中添加监控（Sentry 9.x API）
async function submitOrder(orderData) {
  return await Sentry.startSpan(
    {
      name: 'Submit Order',
      op: 'business_operation'
    },
    async (span) => {
      try {
        addBreadcrumb('开始提交订单', 'business', 'info')

        // 验证订单数据
        await Sentry.startSpan(
          {
            name: 'Validate order data',
            op: 'validation'
          },
          () => {
            validateOrderData(orderData)
          }
        )

        // 提交订单
        const result = await Sentry.startSpan(
          {
            name: 'Submit order to API',
            op: 'api_call'
          },
          () => {
            return api.post('/orders', orderData)
          }
        )

        addBreadcrumb('订单提交成功', 'business', 'info')
        span.setStatus('ok')

        return result.data
      } catch (error) {
        addBreadcrumb('订单提交失败', 'business', 'error')
        span.setStatus('internal_error')

        Sentry.withScope((scope) => {
          scope.setTag('business_operation', 'submit_order')
          scope.setContext('order_data', orderData)
          Sentry.captureException(error)
        })

        throw error
      }
    }
  )
}
```

## 6. Session Replay功能

### 6.1 什么是Session Replay

Session Replay（会话回放）是Sentry 9.x版本的强大新功能，它可以：

- 🎥 **录制用户会话**：像录制视频一样记录用户在浏览器中的操作
- 🐛 **错误重现**：当错误发生时，可以看到用户具体操作了什么
- 📊 **性能分析**：结合性能数据，了解用户体验问题
- 🔒 **隐私保护**：自动隐藏敏感信息，保护用户隐私

### 6.2 基础配置

#### 启用Session Replay

在Sentry配置中已经包含了基础的Session Replay设置：

```javascript
// 在setupSentry函数中
Sentry.init({
  // ... 其他配置

  // Session Replay采样率
  replaysSessionSampleRate: 0.1, // 10%的会话进行录制
  replaysOnErrorSampleRate: 1.0, // 发生错误时100%录制

  integrations: [
    Sentry.replayIntegration({
      // 隐私保护设置
      maskAllText: true,        // 隐藏所有文本
      blockAllMedia: true,      // 阻止媒体文件
      maskAllInputs: true,      // 隐藏输入框内容

      // 网络请求记录
      networkDetailAllowUrls: [], // 允许详细记录的URL
      networkCaptureBodies: false, // 不记录请求体
    }),
  ],
})
```

#### 采样策略说明

- **replaysSessionSampleRate**: 会话级别采样
  - `0.1` = 10%的用户会话被录制
  - `1.0` = 100%的用户会话被录制（开发环境推荐）
  - `0.0` = 禁用会话录制

- **replaysOnErrorSampleRate**: 错误时采样
  - `1.0` = 发生错误时100%录制（推荐设置）
  - 会追溯录制错误前60秒的操作

### 6.3 隐私保护配置

#### 基础隐私设置

```javascript
Sentry.replayIntegration({
  // 文本内容处理
  maskAllText: true,          // 隐藏所有文本（推荐开启）
  maskTextSelector: '.sensitive', // 只隐藏特定选择器的文本

  // 输入框处理
  maskAllInputs: true,        // 隐藏所有输入框（推荐开启）
  maskInputOptions: {
    email: true,             // 隐藏邮箱输入
    password: true,          // 隐藏密码输入
    textarea: true,          // 隐藏文本域
  },

  // 媒体文件处理
  blockAllMedia: true,        // 阻止所有媒体文件（推荐开启）
  blockSelector: '.private-content', // 阻止特定选择器的内容

  // 忽略特定元素
  ignoreSelector: '.no-replay', // 完全忽略的元素
})
```

#### 高级隐私配置

```javascript
Sentry.replayIntegration({
  // 自定义遮罩函数
  maskTextFn: (text, element) => {
    // 自定义文本遮罩逻辑
    if (element.classList.contains('user-id')) {
      return 'USER_ID_MASKED'
    }
    return text.replace(/\d{4}-\d{4}-\d{4}-\d{4}/g, 'CARD_NUMBER_MASKED')
  },

  // 网络请求过滤
  networkDetailAllowUrls: [
    /^\/api\/public/, // 只记录公开API的详细信息
  ],

  // 自定义网络遮罩
  beforeAddNetworkEvent: (event) => {
    // 移除敏感请求头
    if (event.data.request?.headers) {
      delete event.data.request.headers.authorization
      delete event.data.request.headers['x-api-key']
    }
    return event
  },
})
```

### 6.4 手动控制录制

#### 获取Replay实例

```javascript
// 获取当前的Replay实例
const replay = Sentry.getReplay()

// 检查录制状态
console.log('录制状态:', replay.isEnabled())
console.log('Replay ID:', replay.getReplayId())
```

#### 手动开始/停止录制

```javascript
// 手动开始会话录制
replay.start()

// 开始缓冲模式（仅在错误时上传）
replay.startBuffering()

// 停止录制
await replay.stop()

// 立即上传当前录制内容
await replay.flush()
```

#### 条件控制录制

```javascript
// 根据用户类型控制录制
function setupConditionalReplay() {
  const user = getCurrentUser()

  if (user.role === 'admin' || user.isPremium) {
    // VIP用户100%录制
    const replay = Sentry.getReplay()
    replay.start()
  }
}

// 根据页面路径控制录制
function setupPageBasedReplay() {
  const router = useRouter()

  router.afterEach((to) => {
    const replay = Sentry.getReplay()

    if (to.path.startsWith('/admin')) {
      // 管理页面开始录制
      replay.start()
    } else if (to.path === '/checkout') {
      // 结账页面强制录制
      replay.flush()
    }
  })
}
```

### 6.5 错误关联配置

#### 错误采样控制

```javascript
Sentry.replayIntegration({
  // 自定义错误采样逻辑
  beforeErrorSampling: (event) => {
    // 跳过特定类型的错误
    if (event.exception?.values?.[0]?.value?.includes('Network Error')) {
      return false // 不为网络错误录制Replay
    }

    // 为严重错误强制录制
    if (event.level === 'fatal') {
      return true
    }

    return true // 其他错误按正常采样率处理
  },
})
```

#### 添加自定义上下文

```javascript
// 在关键操作前添加Replay标记
function handleCriticalOperation() {
  const replay = Sentry.getReplay()

  if (replay.isEnabled()) {
    // 添加自定义事件到Replay时间线
    Sentry.addBreadcrumb({
      category: 'user.action',
      message: '用户开始执行关键操作',
      level: 'info',
      data: {
        operation: 'payment-process',
        amount: 100,
      }
    })
  }

  try {
    // 执行关键操作
    performPayment()
  } catch (error) {
    // 错误会自动关联到Replay
    Sentry.captureException(error)
  }
}
```

### 6.6 Canvas录制（可选）

如果应用包含Canvas元素，可以启用Canvas录制：

```javascript
// 在integrations数组中添加
integrations: [
  Sentry.replayIntegration(/* ... */),

  // Canvas录制集成
  Sentry.replayCanvasIntegration({
    enableManualSnapshot: false, // 自动截图
    fps: 2, // 每秒2帧
    quality: 'medium', // 图片质量
  }),
]
```

### 6.7 性能优化

#### CSP配置

Session Replay使用Web Worker，需要配置CSP：

```html
<!-- 在HTML头部添加 -->
<meta http-equiv="Content-Security-Policy"
      content="worker-src 'self' blob:; child-src 'self' blob:;">
```

**注意**：
- `worker-src 'self' blob:` - 支持现代浏览器
- `child-src 'self' blob:` - 兼容Safari <= 15.4

如果你无法更新CSP策略，可以使用自定义压缩Worker：

```javascript
Sentry.replayIntegration({
  // 使用自托管的Worker脚本
  workerUrl: '/assets/sentry-replay-worker.js',
})
```

#### 懒加载配置

```javascript
// 延迟加载Replay功能
async function loadReplayLater() {
  // 初始化时不包含Replay
  Sentry.init({
    dsn: 'YOUR_DSN',
    integrations: [
      // 其他集成，但不包含replayIntegration
    ],
  })

  // 在需要时动态添加
  const { replayIntegration } = await import('@sentry/vue')
  Sentry.addIntegration(replayIntegration({
    // 配置选项
  }))
}
```

## 7. 性能监控配置

### 7.1 浏览器性能追踪

```javascript
Sentry.browserTracingIntegration({
  // Vue Router集成
  router,

  // 组件追踪
  trackComponents: true,           // 追踪Vue组件
  timeout: 5000,                  // 事务超时时间

  // 网络请求追踪
  traceFetch: true,               // 追踪fetch请求
  traceXHR: true,                 // 追踪XMLHttpRequest

  // 传播配置
  tracePropagationTargets: [
    'localhost',
    'your-api-domain.com',
    /^https:\/\/api\.yoursite\.com/,
  ],

  // Web Vitals
  enableLongTask: true,           // 启用长任务监控
  enableInp: true,                // 启用交互延迟监控
})
```

### 7.2 自定义性能标记

```javascript
import * as Sentry from '@sentry/vue'

// 手动创建性能事务
function trackCustomOperation() {
  return Sentry.startSpan({
    name: 'custom-operation',
    op: 'function',
  }, async (span) => {
    // 执行操作
    const result = await performHeavyOperation()

    // 添加标签和数据
    span.setTag('operation.type', 'heavy-computation')
    span.setData('result.size', result.length)

    return result
  })
}

// API请求性能追踪
async function trackApiCall(url, data) {
  return Sentry.startSpan({
    name: `API ${url}`,
    op: 'http.client',
    data: {
      url,
      method: 'POST',
    }
  }, async (span) => {
    try {
      const response = await fetch(url, {
        method: 'POST',
        body: JSON.stringify(data),
      })

      span.setTag('http.status_code', response.status)
      span.setData('response.size', response.headers.get('content-length'))

      return response.json()
    } catch (error) {
      span.setTag('error', true)
      throw error
    }
  })
}
```

## 8. Sentry后台使用说明

### 8.1 仪表板导航

#### 主要菜单结构
- **Issues（问题）**: 错误和异常管理
- **Performance（性能）**: 性能监控和分析
- **Releases（发布）**: 版本管理和部署跟踪
- **Alerts（告警）**: 告警规则和通知设置
- **Settings（设置）**: 项目和组织配置

### 8.2 问题管理（Issues）

#### 问题列表界面
- **状态筛选**: Unresolved（未解决）、Resolved（已解决）、Ignored（已忽略）
- **排序选项**:
  - Priority（优先级）
  - Last Seen（最后出现时间）
  - First Seen（首次出现时间）
  - Frequency（频率）
- **时间范围**: 24h、7d、14d、30d或自定义时间范围

#### 问题详情页面
1. **错误概览**
   - 错误标题和描述
   - 发生次数和影响用户数
   - 首次/最后出现时间
   - 错误趋势图

2. **堆栈跟踪（Stack Trace）**
   - 完整的错误堆栈信息
   - 源码定位（如果上传了Source Maps）
   - 相关文件和行号

3. **面包屑（Breadcrumbs）**
   - 错误发生前的用户操作路径
   - API请求记录
   - 页面导航历史

4. **用户上下文**
   - 用户ID、邮箱等信息
   - 浏览器信息
   - 设备信息
   - IP地址和地理位置

5. **标签和环境**
   - 自定义标签
   - 环境信息（development、production等）
   - 发布版本

#### 问题操作
- **Resolve（标记解决）**: 标记问题已修复
- **Ignore（忽略）**: 忽略此问题（可设置忽略条件）
- **Assign（分配）**: 分配给团队成员处理
- **Bookmark（收藏）**: 收藏重要问题
- **Merge（合并）**: 合并重复问题

### 8.3 性能监控（Performance）

#### 性能概览
- **Apdex分数**: 应用性能指数
- **吞吐量**: 每分钟事务数
- **平均响应时间**: 各类操作的平均耗时
- **错误率**: 事务错误率

#### 事务详情
- **事务列表**: 各类操作的性能数据
- **Web Vitals**: 核心网页指标（LCP、FID、CLS）
- **分布图**: 响应时间分布
- **热力图**: 性能热点分析

### 8.4 发布管理（Releases）

#### 创建发布
```bash
# 使用Sentry CLI创建发布
sentry-cli releases new "1.0.0"

# 上传Source Maps
sentry-cli releases files "1.0.0" upload-sourcemaps ./dist

# 完成发布
sentry-cli releases finalize "1.0.0"
```

#### 发布功能
- **版本对比**: 对比不同版本的错误率和性能
- **部署跟踪**: 跟踪部署进度和影响
- **回滚检测**: 自动检测问题版本
- **Source Map管理**: 自动解析压缩代码的错误堆栈

### 8.5 告警设置（Alerts）

#### 告警规则类型
1. **Issue Alerts（问题告警）**
   - 新问题出现时告警
   - 问题频率超过阈值时告警
   - 特定条件匹配时告警

2. **Metric Alerts（指标告警）**
   - 错误率超过阈值
   - 性能指标异常
   - 自定义指标告警

#### 通知方式配置
- **邮件通知**: 配置SMTP服务器
- **Slack集成**: 发送到Slack频道
- **钉钉集成**: 企业微信/钉钉机器人
- **Webhook**: 自定义Webhook通知

### 8.6 团队协作

#### 用户管理
- **邀请成员**: 通过邮箱邀请团队成员
- **权限控制**: 设置不同角色权限
- **团队分组**: 创建不同项目团队

#### 工作流配置
- **代码集成**: GitHub、GitLab集成
- **CI/CD集成**: Jenkins、GitHub Actions等
- **第三方工具**: Jira、Trello等项目管理工具

## 9. 参数详解（Sentry 9.x版本）

### 9.1 Sentry.init() 配置参数

#### 基础配置
```javascript
{
  // 数据源名称，用于标识项目和认证
  dsn: 'string', // 必需

  // 调试模式，开启后会在控制台输出详细信息
  debug: boolean, // 默认: false

  // 环境标识，用于区分开发、测试、生产环境
  environment: 'string', // 默认: 'production'

  // 发布版本号，用于版本对比和回滚检测
  release: 'string', // 默认: undefined

  // 服务器名称，用于标识不同的服务器实例
  serverName: 'string', // 默认: undefined

  // 是否启用自动会话跟踪
  autoSessionTracking: boolean, // 默认: true

  // 最大面包屑数量
  maxBreadcrumbs: number, // 默认: 100
}
```

#### 采样配置
```javascript
{
  // 错误采样率 (0.0 - 1.0)
  sampleRate: number, // 默认: 1.0

  // 性能监控采样率 (0.0 - 1.0)
  tracesSampleRate: number, // 默认: undefined

  // 动态采样函数
  tracesSampler: (samplingContext) => number,

  // 性能监控采样函数
  beforeSend: (event, hint) => event | null,

  // 事务采样函数
  beforeSendTransaction: (event, hint) => event | null,
}
```

#### 集成配置
```javascript
{
  // 集成插件列表
  integrations: [
    // 浏览器追踪集成
    new BrowserTracing({
      // 路由变化追踪
      routingInstrumentation: Sentry.vueRouterInstrumentation(router),

      // 自动追踪的目标
      tracePropagationTargets: ['localhost', 'api.example.com'],

      // 是否追踪组件更新
      trackComponents: boolean, // 默认: false

      // 超时时间（毫秒）
      idleTimeout: number, // 默认: 1000

      // 最终超时时间（毫秒）
      finalTimeout: number, // 默认: 30000
    }),

    // 重放集成（用户会话录制）
    new Replay({
      // 采样率
      sessionSampleRate: number, // 默认: 0.1

      // 错误会话采样率
      errorSampleRate: number, // 默认: 1.0

      // 是否屏蔽敏感信息
      maskAllText: boolean, // 默认: false

      // 是否屏蔽所有输入
      maskAllInputs: boolean, // 默认: false
    })
  ],

  // 默认集成开关
  defaultIntegrations: boolean, // 默认: true

  // 禁用特定集成
  integrations: (integrations) => integrations.filter(/* 过滤逻辑 */),
}
```

### 9.2 错误处理配置

#### beforeSend 函数参数
```javascript
beforeSend(event, hint) {
  // event: Sentry事件对象
  {
    event_id: 'string',        // 事件唯一ID
    timestamp: number,         // 事件时间戳
    level: 'error|warning|info|debug', // 错误级别
    message: 'string',         // 错误消息
    platform: 'javascript',   // 平台标识
    environment: 'string',     // 环境标识
    release: 'string',         // 版本号
    user: {                    // 用户信息
      id: 'string',
      email: 'string',
      username: 'string'
    },
    tags: {},                  // 标签
    contexts: {},              // 上下文信息
    breadcrumbs: [],           // 面包屑
    exception: {               // 异常信息
      values: [{
        type: 'string',        // 异常类型
        value: 'string',       // 异常消息
        stacktrace: {}         // 堆栈信息
      }]
    },
    request: {                 // 请求信息
      url: 'string',
      method: 'string',
      headers: {},
      data: {}
    }
  }

  // hint: 额外信息
  {
    originalException: Error,  // 原始异常对象
    syntheticException: Error, // 合成异常对象
    eventId: 'string',        // 事件ID
    captureContext: {}         // 捕获上下文
  }

  // 返回值
  return event;     // 发送事件
  return null;      // 丢弃事件
  return modified;  // 发送修改后的事件
}
```

### 9.3 性能监控配置

#### BrowserTracing 参数详解
```javascript
new BrowserTracing({
  // 追踪传播目标（哪些请求需要添加追踪头）
  tracePropagationTargets: [
    'localhost',
    'api.example.com',
    /^https:\/\/api\.myapp\.com/
  ],

  // 路由追踪配置
  routingInstrumentation: Sentry.vueRouterInstrumentation(router, {
    // 是否追踪组件更新
    trackComponents: true,

    // 超时配置
    timeout: 5000,

    // 是否包含查询参数
    includeQuery: true
  }),

  // 空闲超时（毫秒）
  idleTimeout: 1000,

  // 最终超时（毫秒）
  finalTimeout: 30000,

  // 心跳间隔（毫秒）
  heartbeatInterval: 5000,

  // 是否创建导航事务
  startTransactionOnLocationChange: true,

  // 是否创建页面加载事务
  startTransactionOnPageLoad: true,

  // 是否追踪长任务
  enableLongTask: true,

  // 是否追踪交互事件
  enableInteractionTracing: true,

  // 最大事务持续时间（秒）
  maxTransactionDuration: 600,

  // 是否屏蔽URL中的敏感信息
  beforeNavigate: (context) => {
    return {
      ...context,
      name: context.name.replace(/\/users\/\d+/, '/users/[id]')
    }
  }
})
```

### 9.4 标签和上下文

#### 标签（Tags）
```javascript
// 设置标签
Sentry.setTag('key', 'value');
Sentry.setTags({
  'user_type': 'premium',
  'feature_flag': 'enabled',
  'component': 'checkout'
});

// 标签用途：
// - 过滤和搜索
// - 统计分析
// - 告警规则
// - 简单的键值对
```

#### 上下文（Contexts）
```javascript
// 设置上下文
Sentry.setContext('user_preferences', {
  theme: 'dark',
  language: 'zh-CN',
  timezone: 'Asia/Shanghai'
});

Sentry.setContext('business_context', {
  order_id: '12345',
  product_category: 'electronics',
  payment_method: 'credit_card'
});

// 上下文用途：
// - 结构化数据
// - 复杂对象信息
// - 业务相关数据
// - 调试信息
```

### 9.5 面包屑配置

```javascript
// 手动添加面包屑
Sentry.addBreadcrumb({
  message: '用户点击购买按钮',
  category: 'ui',
  level: 'info',
  timestamp: Date.now() / 1000,
  data: {
    product_id: '12345',
    quantity: 2
  }
});

// 面包屑类型
{
  category: 'string',  // 分类: http, navigation, ui, console等
  level: 'string',     // 级别: fatal, error, warning, info, debug
  message: 'string',   // 消息内容
  timestamp: number,   // 时间戳
  type: 'string',      // 类型: default, http, error等
  data: object         // 附加数据
}
```

### 9.6 replayIntegration() 配置参数

#### 隐私保护配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `maskAllText` | boolean | true | 隐藏所有文本内容 |
| `maskAllInputs` | boolean | true | 隐藏所有输入框内容 |
| `blockAllMedia` | boolean | true | 阻止所有媒体文件 |
| `maskTextSelector` | string | - | 指定隐藏文本的CSS选择器 |
| `blockSelector` | string | - | 指定阻止的CSS选择器 |
| `ignoreSelector` | string | - | 指定忽略的CSS选择器 |

#### 网络请求配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `networkDetailAllowUrls` | array | [] | 允许记录详细信息的URL |
| `networkCaptureBodies` | boolean | false | 是否捕获请求体 |
| `networkRequestHeaders` | array | [] | 记录的请求头 |
| `networkResponseHeaders` | array | [] | 记录的响应头 |

#### 高级配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `beforeErrorSampling` | function | - | 错误采样前的钩子函数 |
| `beforeAddNetworkEvent` | function | - | 添加网络事件前的钩子函数 |
| `maskTextFn` | function | - | 自定义文本遮罩函数 |

### 9.7 browserTracingIntegration() 配置参数

#### 路由追踪配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `router` | object | - | Vue Router实例 |
| `trackComponents` | boolean | false | 是否追踪Vue组件 |
| `timeout` | number | 1000 | 事务超时时间（毫秒） |

#### 网络追踪配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `traceFetch` | boolean | true | 是否追踪fetch请求 |
| `traceXHR` | boolean | true | 是否追踪XMLHttpRequest |
| `tracePropagationTargets` | array | - | 分布式追踪的目标URL |

#### Web Vitals配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enableLongTask` | boolean | true | 启用长任务监控 |
| `enableInp` | boolean | true | 启用交互延迟监控 |

### 9.8 Sentry 9.x版本重要变更

#### 已废弃的参数

| 废弃参数 | 替代方案 | 说明 |
|----------|----------|------|
| `enableTracing` | `tracesSampleRate: 1.0` | 使用采样率控制 |
| `autoSessionTracking` | - | 自动启用，无需配置 |
| `transactionContext` | `samplingContext` | 采样上下文重构 |

#### 新增的Hook函数

| Hook函数 | 说明 | 使用场景 |
|----------|------|----------|
| `beforeSendSpan` | 发送span前的钩子 | 修改或过滤性能数据 |
| `beforeSendTransaction` | 发送事务前的钩子 | 修改或过滤事务数据 |

#### 集成语法变更

```javascript
// Sentry 8.x（旧语法）
import { BrowserTracing } from '@sentry/tracing'

integrations: [
  new BrowserTracing({
    routingInstrumentation: Sentry.vueRouterInstrumentation(router),
  })
]

// Sentry 9.x（新语法）
integrations: [
  Sentry.browserTracingIntegration({
    router, // 直接传入router实例
  })
]
```

## 10. 常见问题与解决方案

### 10.1 Sentry 9.x升级问题

**Q: 从8.x升级到9.x后出现TypeError: Sentry.replayIntegration is not a function**

A: 这是因为集成语法发生了变化，检查以下几点：
1. 确保安装了最新的Sentry 9.x版本：
   ```bash
   npm install @sentry/vue@^9.0.0
   ```
2. 移除旧的`@sentry/tracing`依赖：
   ```bash
   npm uninstall @sentry/tracing
   ```
3. 更新集成语法：
   ```javascript
   // 新语法
   integrations: [
     Sentry.browserTracingIntegration({ router }),
     Sentry.replayIntegration(),
   ]
   ```

**Q: Session Replay功能无法正常工作**

A: 检查以下配置：
1. 确认采样率设置正确：
   ```javascript
   replaysSessionSampleRate: 1.0, // 开发环境设为1.0
   replaysOnErrorSampleRate: 1.0,
   ```
2. 检查CSP配置是否允许Web Worker：
   ```html
   <meta http-equiv="Content-Security-Policy"
         content="worker-src 'self' blob:;">
   ```
3. 确认浏览器兼容性（需要支持ES2020）

**Q: 性能监控数据不准确**

A:
1. 确认`tracesSampleRate`设置正确
2. 检查`tracePropagationTargets`配置
3. 验证路由集成是否正确传入router实例

### 10.2 安装和配置问题

#### Q1: DSN配置错误
**问题**: 控制台提示 "Invalid DSN"
**解决方案**:
```javascript
// 确保DSN格式正确
// 正确格式: http://PUBLIC_KEY@localhost:9000/PROJECT_ID
// 检查项目ID和公钥是否正确

// 在Sentry后台获取正确的DSN
// 项目设置 -> Client Keys (DSN) -> 复制DSN
```

#### Q2: Source Maps上传失败
**问题**: 生产环境看不到源码定位
**解决方案**:
```bash
# 安装Sentry CLI
npm install -g @sentry/cli

# 配置认证
echo "defaults.url=http://localhost:9000
defaults.org=your-org
defaults.project=vue3-frontend
auth.token=YOUR_AUTH_TOKEN" > ~/.sentryclirc

# 上传Source Maps
sentry-cli releases files VERSION upload-sourcemaps ./dist --url-prefix "~/static/js/"
```

#### Q3: 本地开发环境报错过多
**解决方案**:
```javascript
// 开发环境配置
Sentry.init({
  dsn: process.env.VUE_APP_SENTRY_DSN,
  environment: 'development',
  sampleRate: 0.1, // 降低采样率
  beforeSend(event) {
    // 开发环境只在控制台显示，不上报
    if (process.env.NODE_ENV === 'development') {
      console.error('Dev Error:', event);
      return null; // 不上报到Sentry
    }
    return event;
  }
});
```

### 10.3 性能监控问题

#### Q4: 性能数据采集过多
**解决方案**:
```javascript
Sentry.init({
  // 降低性能监控采样率
  tracesSampleRate: 0.1, // 只采集10%的事务

  // 或使用动态采样
  tracesSampler: (samplingContext) => {
    if (samplingContext.transactionContext.name === 'critical_page') {
      return 1.0; // 关键页面100%采样
    }
    return 0.1; // 其他页面10%采样
  }
});
```

#### Q5: 路由追踪不准确
**解决方案**:
```javascript
// 确保正确配置路由追踪
new BrowserTracing({
  routingInstrumentation: Sentry.vueRouterInstrumentation(router, {
    trackComponents: true,
    timeout: 5000
  }),
  beforeNavigate: (context) => {
    // 规范化路由名称
    return {
      ...context,
      name: context.name.replace(/\/\d+/g, '/[id]')
    };
  }
});
```

### 10.4 数据隐私和安全

#### Q6: 敏感信息泄露
**解决方案**:
```javascript
// 过滤敏感信息
Sentry.init({
  beforeSend(event) {
    // 移除敏感的请求数据
    if (event.request) {
      delete event.request.cookies;

      // 过滤敏感字段
      if (event.request.data) {
        ['password', 'token', 'credit_card'].forEach(field => {
          if (event.request.data[field]) {
            event.request.data[field] = '[Filtered]';
          }
        });
      }
    }

    // 过滤用户敏感信息
    if (event.user) {
      delete event.user.ip_address;
    }

    return event;
  }
});
```

### 10.5 告警和通知

#### Q7: 告警过于频繁
**解决方案**:
1. 在Sentry后台设置告警规则
2. 调整触发频率阈值
3. 设置告警静默期
4. 配置告警优先级

#### Q8: 邮件通知配置
**解决方案**:
```bash
# 在Sentry后台配置SMTP
# Settings -> Mail -> SMTP Settings

# 或在docker-compose.yml中配置
environment:
  SENTRY_EMAIL_HOST: smtp.gmail.com
  SENTRY_EMAIL_PORT: 587
  SENTRY_EMAIL_USE_TLS: true
  SENTRY_EMAIL_HOST_USER: your-email@gmail.com
  SENTRY_EMAIL_HOST_PASSWORD: your-app-password
```

## 11. 版本升级指导

### 11.1 从Sentry 8.x升级到9.x

#### 主要变更概述

| 变更项 | Sentry 8.x | Sentry 9.x | 影响 |
|--------|------------|------------|------|
| 包依赖 | 需要`@sentry/tracing` | 集成到主包 | 简化依赖管理 |
| 集成语法 | `new Integration()` | `Sentry.integration()` | 函数式调用 |
| Session Replay | 实验性 | 生产可用 | 增强错误调试 |
| Bundle大小 | 较大 | 优化减小 | 性能提升 |

#### 升级步骤

1. **更新依赖包**：
   ```bash
   # 移除旧的tracing包
   npm uninstall @sentry/tracing

   # 更新主包到9.x
   npm install @sentry/vue@^9.0.0
   ```

2. **更新导入语句**：
   ```javascript
   // 删除以下导入（9.x不再需要）
   // import { BrowserTracing } from '@sentry/tracing'
   // import { Replay } from '@sentry/replay'

   // 只需要主包导入
   import * as Sentry from '@sentry/vue'
   ```

3. **更新配置语法**：
   ```javascript
   // 旧语法（8.x）
   Sentry.init({
     integrations: [
       new BrowserTracing({
         routingInstrumentation: Sentry.vueRouterInstrumentation(router),
       }),
       new Replay({
         sessionSampleRate: 0.1,
       }),
     ],
     enableTracing: true, // 已废弃
   })

   // 新语法（9.x）
   Sentry.init({
     integrations: [
       Sentry.browserTracingIntegration({
         router, // 直接传入router实例
       }),
       Sentry.replayIntegration(),
     ],
     tracesSampleRate: 1.0, // 替代enableTracing
     replaysSessionSampleRate: 0.1, // 直接在init中配置
     replaysOnErrorSampleRate: 1.0,
   })
   ```

#### 配置参数映射

| 8.x参数 | 9.x参数 | 说明 |
|---------|---------|------|
| `enableTracing` | `tracesSampleRate` | 使用采样率控制 |
| `BrowserTracing` | `browserTracingIntegration` | 函数式集成 |
| `Replay` | `replayIntegration` | 内置集成 |

### 11.2 新特性使用指南

#### Session Replay增强功能

```javascript
// 高级Replay配置
Sentry.replayIntegration({
  // 隐私保护设置
  maskAllText: true,
  maskAllInputs: true,
  blockAllMedia: true,

  // Canvas录制（新特性）
  recordCanvas: true,

  // 网络请求过滤（增强）
  networkDetailAllowUrls: [
    /^\/api\/public/,
    /^\/api\/safe/,
  ],
  networkCaptureBodies: false,

  // 错误时会话采样配置在init中设置
  // replaysOnErrorSampleRate: 1.0 // 在Sentry.init()中配置
})
```

#### 性能监控优化

```javascript
// 新的性能监控特性
Sentry.browserTracingIntegration({
  // Web Vitals增强
  enableLongTask: true,
  enableInp: true, // 交互延迟监控

  // 自定义instrumentation
  instrumentPageLoad: true,
  instrumentNavigation: true,

  // 更精确的采样控制
  beforeStartSpan: (options) => {
    // 动态调整span配置
    if (options.name.includes('api')) {
      options.attributes = {
        ...options.attributes,
        'custom.api_version': 'v2',
      }
    }
    return options
  },
})
```

### 11.3 兼容性检查清单

#### 必检项目

- [ ] 移除`@sentry/tracing`依赖
- [ ] 更新集成语法为函数调用
- [ ] 替换废弃的配置参数
- [ ] 测试Session Replay功能
- [ ] 验证性能监控数据
- [ ] 检查错误上报正常

#### 可选优化

- [ ] 启用新的Web Vitals监控
- [ ] 配置Canvas录制（如适用）
- [ ] 优化Replay采样策略
- [ ] 更新CSP头支持Web Worker
- [ ] 配置新的告警规则

## 12. 最佳实践总结

### 12.1 配置建议
- ✅ 生产环境使用适当的采样率（10-20%）
- ✅ 配置合适的错误过滤规则
- ✅ 设置有意义的标签和上下文
- ✅ 上传Source Maps以便调试
- ✅ 配置版本发布跟踪

### 12.2 监控策略
- ✅ 关键业务流程全链路监控
- ✅ API请求错误自动上报
- ✅ 用户操作路径跟踪
- ✅ 性能指标阈值告警
- ✅ 定期查看和处理错误

### 12.3 团队协作
- ✅ 制定错误处理流程
- ✅ 分配错误负责人
- ✅ 定期回顾错误趋势
- ✅ 建立发布质量监控
- ✅ 培训团队使用Sentry

## 13. 版本管理和更新策略

### 13.1 版本选择策略

#### 为什么需要版本管理策略？

Sentry更新频繁，生产环境需要稳定性。盲目跟随最新版本可能带来风险：
- 🔴 新版本可能有未知bug
- 🟡 API变更可能影响现有功能
- 🟢 频繁更新增加维护成本

#### 推荐的版本选择原则

```json
{
  "dependencies": {
    "@sentry/vue": "~9.15.0",  // ✅ 锁定小版本，只允许patch更新
    // 而不是 "^9.15.0" (❌ 允许minor更新，风险较高)
  }
}
```

**版本选择建议**：
```
📅 更新频率建议：
- 🔴 大版本更新：6-12个月一次（充分测试后）
- 🟡 小版本更新：2-3个月一次（评估后）
- 🟢 补丁更新：随时（安全修复优先）

🎯 版本选择策略：
- ✅ 选择发布3-6个月的版本（经过社区验证）
- ✅ 关注LTS或标记为稳定的版本
- ❌ 避免使用刚发布的最新版本
```

### 13.2 分层更新策略

#### 环境分层管理

```
开发环境 → 测试环境 → 预发布环境 → 生产环境
  ↓           ↓           ↓            ↓
 最新版本    稳定版本    验证版本     保守版本
 (验证新功能) (功能测试)  (性能确认)   (稳定运行)
```

#### 实际配置示例

```javascript
// config/sentry.config.js
const sentryVersions = {
  development: '~9.30.0',    // 开发环境：较新版本，验证新功能
  testing: '~9.25.0',       // 测试环境：稳定版本
  staging: '~9.20.0',       // 预发布：验证过的版本
  production: '~9.15.0'     // 生产环境：保守版本
}

const currentEnv = process.env.NODE_ENV || 'development'
export const SENTRY_VERSION = sentryVersions[currentEnv]
```

### 13.3 版本更新检查清单

#### 更新前评估清单

```markdown
## Sentry 版本更新检查清单

### 📋 更新前评估
- [ ] 查看官方 CHANGELOG 了解破坏性变更
- [ ] 检查是否有关键安全漏洞修复
- [ ] 评估新功能是否对项目有实际价值
- [ ] 确认团队有足够时间进行测试
- [ ] 检查依赖包兼容性

### 🧪 更新测试流程
- [ ] 在开发环境验证基础功能
- [ ] 测试错误上报是否正常
- [ ] 验证性能监控数据准确性
- [ ] 检查 Session Replay 功能稳定性
- [ ] 确认 Source Maps 正常解析
- [ ] 验证告警和通知功能
- [ ] 测试 API 集成和自动化流程

### 🚀 生产部署步骤
- [ ] 制定详细的发布计划
- [ ] 准备回滚预案和脚本
- [ ] 进行灰度发布验证
- [ ] 实时监控错误率变化
- [ ] 观察性能指标波动
- [ ] 确认用户体验无异常
```

### 13.4 依赖管理最佳实践

#### 锁定依赖版本

```json
// package.json - 推荐配置
{
  "dependencies": {
    "@sentry/vue": "~9.15.0"  // 精确控制版本
  },
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=8.0.0"
  },
  "volta": {
    "node": "18.19.0",
    "npm": "10.2.4"
  }
}
```

#### 使用锁定文件

```bash
# 生成并提交锁定文件
npm install
git add package-lock.json

# 或使用 yarn
yarn install
git add yarn.lock

# CI/CD 中使用锁定安装
npm ci  # 或 yarn install --frozen-lockfile
```

### 13.5 监控版本健康度

#### 版本监控脚本

```javascript
// utils/sentry-health-check.js
import * as Sentry from '@sentry/vue'

export function monitorSentryHealth() {
  const healthMetrics = {
    sdkVersion: Sentry.SDK_VERSION,
    initTime: Date.now(),
    errorCount: 0,
    lastError: null,
    features: {
      errorTracking: true,
      performanceMonitoring: true,
      sessionReplay: true
    }
  }

  // 监控错误上报功能
  const originalCaptureException = Sentry.captureException
  Sentry.captureException = function(...args) {
    healthMetrics.errorCount++
    healthMetrics.lastError = Date.now()
    return originalCaptureException.apply(this, args)
  }

  // 定期检查健康状态
  setInterval(() => {
    console.info('Sentry健康检查:', healthMetrics)

    // 可选：发送到内部监控系统
    if (window.analytics) {
      window.analytics.track('sentry_health_check', healthMetrics)
    }
  }, 300000) // 每5分钟检查一次

  return healthMetrics
}
```

#### 降级策略

```javascript
// utils/sentry-fallback.js
export function initSentryWithFallback(config) {
  try {
    Sentry.init(config)

    // 验证基础功能
    const testError = new Error('Sentry连接测试')
    Sentry.captureException(testError)

    console.info('✅ Sentry初始化成功')
  } catch (error) {
    console.warn('⚠️ Sentry初始化失败，启用降级方案', error)

    // 降级到基础错误收集
    window.addEventListener('error', (event) => {
      const errorData = {
        message: event.message,
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno,
        timestamp: Date.now()
      }

      // 存储到本地或发送到备用服务
      localStorage.setItem('error_' + Date.now(), JSON.stringify(errorData))
    })

    // 提供基础API兼容性
    window.Sentry = {
      captureException: (error) => {
        console.error('捕获到错误:', error)
        // 存储错误信息
      },
      captureMessage: (message) => {
        console.warn('捕获到消息:', message)
      }
    }
  }
}
```

### 13.6 版本升级实战指南

#### 升级准备工作

```bash
# 1. 备份当前配置
cp package.json package.json.backup
cp package-lock.json package-lock.json.backup

# 2. 创建升级分支
git checkout -b upgrade/sentry-9.25.0

# 3. 查看版本变更
npm info @sentry/vue versions --json
npm view @sentry/vue@9.25.0
```

#### 升级执行步骤

```bash
# 1. 更新到目标版本
npm install @sentry/vue@~9.25.0

# 2. 检查依赖冲突
npm audit
npm ls @sentry/vue

# 3. 运行测试套件
npm run test
npm run test:e2e

# 4. 本地验证
npm run dev
npm run build
```

#### 升级后验证

```javascript
// tests/sentry-integration.test.js
describe('Sentry集成测试', () => {
  beforeEach(() => {
    // 重置Sentry状态
    Sentry.getCurrentHub().getClient()?.close()
  })

  it('应该正确初始化Sentry', () => {
    expect(Sentry.SDK_VERSION).toMatch(/^9\.25\./)
    expect(Sentry.getCurrentHub().getClient()).toBeDefined()
  })

  it('应该能够捕获错误', async () => {
    const mockError = new Error('测试错误')
    const eventId = Sentry.captureException(mockError)
    expect(eventId).toBeDefined()
  })

  it('性能监控应该正常工作', () => {
    const span = Sentry.startInactiveSpan({ name: '测试事务' })
    expect(span).toBeDefined()
    span.end()
  })
})
```

### 13.7 长期维护策略

#### 建立版本评估定期流程

```javascript
// scripts/sentry-version-audit.js
const currentVersion = require('@sentry/vue/package.json').version
const packageInfo = require('../package.json')

console.log(`
📊 Sentry版本审计报告
====================
当前版本: ${currentVersion}
上次更新: ${packageInfo.lastSentryUpdate || '未记录'}
使用时长: ${calculateUsageDuration()}

🔍 检查建议:
- 查看最新版本: npm view @sentry/vue version
- 检查安全公告: https://github.com/getsentry/sentry-javascript/security
- 评估新功能价值
- 制定升级计划
`)
```

#### 团队协作规范

```markdown
## Sentry版本管理团队规范

### 🧑‍💻 责任分工
- **前端负责人**: 技术调研、版本评估
- **测试工程师**: 功能验证、性能测试
- **运维工程师**: 生产部署、监控告警
- **产品经理**: 业务影响评估

### 📅 定期Review
- **月度检查**: 评估是否需要patch更新
- **季度评估**: 考虑minor版本升级
- **年度规划**: 制定大版本升级计划

### 📝 文档维护
- 记录每次升级的原因和结果
- 维护版本兼容性文档
- 更新团队知识库
```

## 14. 性能优化指南

### 14.1 SDK包大小优化

#### Tree-shaking优化

```javascript
// ❌ 避免：导入整个SDK包
import * as Sentry from '@sentry/vue'

// ✅ 推荐：按需导入
import { init, captureException, configureScope } from '@sentry/vue'
import { browserTracingIntegration } from '@sentry/vue'
```

#### 懒加载策略

```javascript
// utils/sentry-lazy.js
let sentryPromise = null

export function initSentryLazy() {
  if (!sentryPromise) {
    sentryPromise = import('@sentry/vue').then((Sentry) => {
      Sentry.init({
        // 配置
      })
      return Sentry
    })
  }
  return sentryPromise
}

// 在需要时才加载
export async function captureErrorLazy(error) {
  const Sentry = await initSentryLazy()
  return Sentry.captureException(error)
}
```

#### 条件加载

```javascript
// main.js - 根据环境条件加载
async function initApp() {
  const app = createApp(App)

  // 只在生产环境或需要时加载Sentry
  if (process.env.NODE_ENV === 'production' || process.env.VUE_APP_ENABLE_SENTRY) {
    const { setupSentry } = await import('./utils/sentry')
    setupSentry(app, router)
  }

  app.mount('#app')
}

initApp()
```

### 14.2 运行时性能优化

#### 异步初始化策略

```javascript
// utils/sentry-async.js
export function setupSentryAsync(app, router) {
  // 先提供基础错误收集
  const errorQueue = []
  window.addEventListener('error', (event) => {
    errorQueue.push({
      type: 'error',
      data: event,
      timestamp: Date.now()
    })
  })

  // 异步初始化Sentry
  setTimeout(async () => {
    const Sentry = await import('@sentry/vue')

    Sentry.init({
      // 正常配置
    })

    // 处理排队的错误
    errorQueue.forEach(queuedError => {
      if (queuedError.type === 'error') {
        Sentry.captureException(new Error(queuedError.data.message))
      }
    })

    errorQueue.length = 0 // 清空队列
  }, 100) // 延迟100ms初始化
}
```

#### 性能监控采样优化

```javascript
// 动态采样策略
export function createDynamicSampler() {
  let errorCount = 0
  let lastResetTime = Date.now()

  return (samplingContext) => {
    const now = Date.now()

    // 每小时重置计数
    if (now - lastResetTime > 3600000) {
      errorCount = 0
      lastResetTime = now
    }

    // 根据错误频率调整采样率
    if (errorCount < 10) {
      errorCount++
      return 1.0 // 前10个错误100%采样
    } else if (errorCount < 50) {
      return 0.5 // 接下来50%采样
    } else {
      return 0.1 // 超过阈值后10%采样
    }
  }
}

// 使用动态采样
Sentry.init({
  tracesSampler: createDynamicSampler(),
})
```

#### 内存使用优化

```javascript
// 配置内存友好的选项
Sentry.init({
  // 限制面包屑数量
  maxBreadcrumbs: 50, // 默认100，减少到50

  // 限制额外数据大小
  maxValueLength: 1000, // 限制值的最大长度

  // Session Replay内存优化
  replaysSessionSampleRate: 0.1,
  integrations: [
    Sentry.replayIntegration({
      // 降低录制质量以节省内存
      maskAllText: true,
      blockAllMedia: true,

      // 限制网络请求记录
      networkDetailAllowUrls: [],
      networkCaptureBodies: false,
    }),
  ],

  // 事件过滤器，减少不必要的事件
  beforeSend: (event) => {
    // 过滤重复错误
    if (event.fingerprint && recentFingerprints.has(event.fingerprint[0])) {
      return null
    }

    // 过滤低价值错误
    if (event.exception?.values?.[0]?.value?.includes('Script error')) {
      return null
    }

    return event
  }
})
```

### 14.3 网络性能优化

#### 批量发送策略

```javascript
// 自定义传输层，实现批量发送
class BatchTransport {
  constructor(options) {
    this.options = options
    this.queue = []
    this.timer = null
  }

  send(event) {
    this.queue.push(event)

    // 达到批量阈值或定时发送
    if (this.queue.length >= 10 || !this.timer) {
      this.timer = setTimeout(() => {
        this.flush()
      }, 5000) // 5秒后发送
    }
  }

  flush() {
    if (this.queue.length === 0) return

    const events = [...this.queue]
    this.queue.length = 0

    // 批量发送到Sentry
    fetch(this.options.url, {
      method: 'POST',
      body: JSON.stringify({ events })
    })

    clearTimeout(this.timer)
    this.timer = null
  }
}
```

#### 网络状态适配

```javascript
// 根据网络状态调整行为
export function createNetworkAwareSentry() {
  const connection = navigator.connection || navigator.mozConnection || navigator.webkitConnection

  let samplingRate = 1.0

  if (connection) {
    switch (connection.effectiveType) {
      case 'slow-2g':
      case '2g':
        samplingRate = 0.1 // 慢网络降低采样率
        break
      case '3g':
        samplingRate = 0.5
        break
      case '4g':
      default:
        samplingRate = 1.0
        break
    }
  }

  return {
    tracesSampleRate: samplingRate,
    replaysSessionSampleRate: connection?.effectiveType === '4g' ? 0.1 : 0.05,
  }
}
```

## 15. 成本控制策略

### 15.1 事件配额管理

#### 配额监控

```javascript
// utils/quota-monitor.js
class SentryQuotaMonitor {
  constructor(monthlyLimit = 10000) {
    this.monthlyLimit = monthlyLimit
    this.currentCount = this.loadCurrentCount()
    this.monthStart = this.getMonthStart()
  }

  loadCurrentCount() {
    const stored = localStorage.getItem('sentry_event_count')
    if (!stored) return 0

    const data = JSON.parse(stored)

    // 检查是否是新月份
    if (data.month !== new Date().getMonth()) {
      return 0
    }

    return data.count
  }

  saveCurrentCount() {
    localStorage.setItem('sentry_event_count', JSON.stringify({
      count: this.currentCount,
      month: new Date().getMonth()
    }))
  }

  shouldSendEvent() {
    if (this.currentCount >= this.monthlyLimit) {
      console.warn(`Sentry事件已达月度限制: ${this.monthlyLimit}`)
      return false
    }
    return true
  }

  recordEvent() {
    this.currentCount++
    this.saveCurrentCount()

    // 接近限制时警告
    if (this.currentCount >= this.monthlyLimit * 0.8) {
      console.warn(`Sentry事件使用量警告: ${this.currentCount}/${this.monthlyLimit}`)
    }
  }

  getRemainingQuota() {
    return Math.max(0, this.monthlyLimit - this.currentCount)
  }
}

// 集成到Sentry配置
const quotaMonitor = new SentryQuotaMonitor(10000) // 月度限制10K事件

Sentry.init({
  beforeSend: (event) => {
    if (!quotaMonitor.shouldSendEvent()) {
      return null // 超出配额，不发送
    }

    quotaMonitor.recordEvent()
    return event
  }
})
```

#### 智能采样算法

```javascript
// utils/smart-sampling.js
export class SmartSampler {
  constructor(options = {}) {
    this.baseRate = options.baseRate || 0.1
    this.errorBudget = options.errorBudget || 100 // 每小时错误预算
    this.timeBucket = options.timeBucket || 3600000 // 1小时

    this.errorCounts = new Map()
    this.lastCleanup = Date.now()
  }

  shouldSample(samplingContext) {
    this.cleanup() // 清理过期数据

    const now = Date.now()
    const bucket = Math.floor(now / this.timeBucket)
    const currentErrors = this.errorCounts.get(bucket) || 0

    // 基于错误预算的动态采样
    if (currentErrors < this.errorBudget * 0.5) {
      return 1.0 // 错误较少时100%采样
    } else if (currentErrors < this.errorBudget) {
      return 0.5 // 中等错误量50%采样
    } else {
      return this.baseRate // 超出预算时使用基础采样率
    }
  }

  recordError() {
    const now = Date.now()
    const bucket = Math.floor(now / this.timeBucket)
    const current = this.errorCounts.get(bucket) || 0
    this.errorCounts.set(bucket, current + 1)
  }

  cleanup() {
    const now = Date.now()
    if (now - this.lastCleanup < this.timeBucket) return

    const currentBucket = Math.floor(now / this.timeBucket)

    // 清理2小时前的数据
    for (const [bucket] of this.errorCounts) {
      if (bucket < currentBucket - 2) {
        this.errorCounts.delete(bucket)
      }
    }

    this.lastCleanup = now
  }
}

// 使用智能采样
const smartSampler = new SmartSampler({
  baseRate: 0.1,
  errorBudget: 100
})

Sentry.init({
  tracesSampler: (context) => smartSampler.shouldSample(context),
  beforeSend: (event) => {
    smartSampler.recordError()
    return event
  }
})
```

### 15.2 数据保留优化

#### 事件优先级分类

```javascript
// 根据错误重要性分类
export function classifyEventPriority(event) {
  const { level, tags, user, contexts } = event

  // 高优先级：影响付费用户的严重错误
  if (level === 'fatal' || level === 'error') {
    if (user?.subscription === 'premium' || tags?.component === 'payment') {
      return 'high'
    }
  }

  // 中优先级：影响核心功能的错误
  if (level === 'error' && tags?.core_feature) {
    return 'medium'
  }

  // 低优先级：一般警告和信息
  return 'low'
}

// 基于优先级的采样
Sentry.init({
  beforeSend: (event) => {
    const priority = classifyEventPriority(event)

    // 根据优先级决定是否发送
    switch (priority) {
      case 'high':
        return event // 100%发送
      case 'medium':
        return Math.random() < 0.5 ? event : null // 50%发送
      case 'low':
        return Math.random() < 0.1 ? event : null // 10%发送
      default:
        return null
    }
  }
})
```

### 15.3 成本监控和告警

#### 成本跟踪仪表板

```javascript
// utils/cost-tracker.js
export class SentryCostTracker {
  constructor() {
    this.eventCosts = {
      error: 0.001,      // 每个错误事件成本
      transaction: 0.0005, // 每个性能事件成本
      replay: 0.002      // 每个回放会话成本
    }

    this.dailyBudget = 10 // 每日预算 $10
    this.monthlyBudget = 300 // 月度预算 $300
  }

  trackEvent(eventType) {
    const cost = this.eventCosts[eventType] || 0
    const today = new Date().toDateString()

    // 记录到localStorage
    const dailyData = JSON.parse(localStorage.getItem(`sentry_cost_${today}`) || '{"cost": 0, "events": {}}')
    dailyData.cost += cost
    dailyData.events[eventType] = (dailyData.events[eventType] || 0) + 1

    localStorage.setItem(`sentry_cost_${today}`, JSON.stringify(dailyData))

    // 检查预算
    this.checkBudget(dailyData.cost)
  }

  checkBudget(todayCost) {
    if (todayCost >= this.dailyBudget * 0.8) {
      console.warn(`🚨 Sentry成本警告: 今日已使用 $${todayCost.toFixed(3)}，接近预算 $${this.dailyBudget}`)

      // 可以发送告警到团队
      this.sendCostAlert(todayCost)
    }
  }

  sendCostAlert(cost) {
    // 发送到钉钉/Slack等
    fetch('/api/alerts/sentry-cost', {
      method: 'POST',
      body: JSON.stringify({
        message: `Sentry成本警告: 今日已使用 $${cost.toFixed(3)}`,
        level: 'warning'
      })
    })
  }

  generateCostReport() {
    const last7Days = Array.from({length: 7}, (_, i) => {
      const date = new Date(Date.now() - i * 24 * 60 * 60 * 1000).toDateString()
      const data = JSON.parse(localStorage.getItem(`sentry_cost_${date}`) || '{"cost": 0, "events": {}}')
      return { date, ...data }
    })

    return {
      totalCost: last7Days.reduce((sum, day) => sum + day.cost, 0),
      avgDailyCost: last7Days.reduce((sum, day) => sum + day.cost, 0) / 7,
      projectedMonthlyCost: (last7Days.reduce((sum, day) => sum + day.cost, 0) / 7) * 30,
      details: last7Days
    }
  }
}

// 集成成本跟踪
const costTracker = new SentryCostTracker()

Sentry.init({
  beforeSend: (event) => {
    costTracker.trackEvent('error')
    return event
  },
  beforeSendTransaction: (event) => {
    costTracker.trackEvent('transaction')
    return event
  }
})
```

## 16. 应急处理和容错

### 16.1 服务降级策略

#### 自动降级机制

```javascript
// utils/sentry-fallback.js
export class SentryFallbackManager {
  constructor() {
    this.isHealthy = true
    this.failureCount = 0
    this.maxFailures = 3
    this.healthCheckInterval = 60000 // 1分钟检查一次
    this.localErrorQueue = []
    this.maxQueueSize = 100

    this.startHealthCheck()
  }

  async checkSentryHealth() {
    try {
      // 发送测试事件检查连通性
      const testEvent = new Error('Health Check')
      const eventId = Sentry.captureException(testEvent)

      if (eventId) {
        this.onHealthy()
        return true
      }
    } catch (error) {
      this.onUnhealthy(error)
      return false
    }
  }

  onHealthy() {
    if (!this.isHealthy) {
      console.info('✅ Sentry服务已恢复')
      this.isHealthy = true
      this.failureCount = 0

      // 尝试发送排队的错误
      this.flushLocalQueue()
    }
  }

  onUnhealthy(error) {
    this.failureCount++
    console.warn(`⚠️ Sentry健康检查失败 (${this.failureCount}/${this.maxFailures}):`, error)

    if (this.failureCount >= this.maxFailures && this.isHealthy) {
      console.error('🚨 Sentry服务异常，启用降级模式')
      this.isHealthy = false
    }
  }

  startHealthCheck() {
    setInterval(() => {
      if (!this.isHealthy) {
        this.checkSentryHealth()
      }
    }, this.healthCheckInterval)
  }

  captureErrorFallback(error, context = {}) {
    if (this.isHealthy) {
      try {
        return Sentry.captureException(error)
      } catch (e) {
        this.onUnhealthy(e)
        return this.storeLocalError(error, context)
      }
    } else {
      return this.storeLocalError(error, context)
    }
  }

  storeLocalError(error, context) {
    const errorData = {
      message: error.message,
      stack: error.stack,
      context,
      timestamp: Date.now(),
      url: window.location.href,
      userAgent: navigator.userAgent
    }

    this.localErrorQueue.push(errorData)

    // 限制队列大小
    if (this.localErrorQueue.length > this.maxQueueSize) {
      this.localErrorQueue.shift()
    }

    // 存储到localStorage作为备份
    try {
      localStorage.setItem('sentry_offline_errors', JSON.stringify(this.localErrorQueue.slice(-20)))
    } catch (e) {
      // localStorage可能已满，忽略
    }

    return 'offline_' + Date.now()
  }

  async flushLocalQueue() {
    if (this.localErrorQueue.length === 0) return

    console.info(`📤 尝试发送 ${this.localErrorQueue.length} 个离线错误`)

    for (const errorData of this.localErrorQueue) {
      try {
        Sentry.captureException(new Error(errorData.message), {
          contexts: {
            offline_error: errorData
          }
        })
      } catch (e) {
        console.warn('离线错误发送失败:', e)
        break
      }
    }

    this.localErrorQueue.length = 0
    localStorage.removeItem('sentry_offline_errors')
  }
}

// 使用降级管理器
const fallbackManager = new SentryFallbackManager()

// 替换全局错误捕获
window.captureException = (error, context) => {
  return fallbackManager.captureErrorFallback(error, context)
}
```

### 16.2 本地缓存机制

#### 离线错误收集

```javascript
// utils/offline-error-collector.js
export class OfflineErrorCollector {
  constructor() {
    this.storageKey = 'sentry_offline_errors'
    this.maxStorage = 100
    this.syncInterval = 30000 // 30秒尝试同步一次

    this.startSyncProcess()
    this.recoverStoredErrors()
  }

  collectError(error, context = {}) {
    const errorRecord = {
      id: this.generateId(),
      error: {
        message: error.message,
        stack: error.stack,
        name: error.name
      },
      context,
      metadata: {
        timestamp: Date.now(),
        url: window.location.href,
        userAgent: navigator.userAgent,
        viewport: {
          width: window.innerWidth,
          height: window.innerHeight
        }
      }
    }

    this.storeError(errorRecord)
    return errorRecord.id
  }

  storeError(errorRecord) {
    try {
      const stored = this.getStoredErrors()
      stored.push(errorRecord)

      // 限制存储数量
      if (stored.length > this.maxStorage) {
        stored.splice(0, stored.length - this.maxStorage)
      }

      localStorage.setItem(this.storageKey, JSON.stringify(stored))
    } catch (e) {
      console.warn('无法存储离线错误:', e)

      // 如果localStorage满了，尝试清理旧数据
      this.cleanupOldErrors()
    }
  }

  getStoredErrors() {
    try {
      const stored = localStorage.getItem(this.storageKey)
      return stored ? JSON.parse(stored) : []
    } catch (e) {
      return []
    }
  }

  async syncStoredErrors() {
    const errors = this.getStoredErrors()
    if (errors.length === 0) return

    const syncedIds = []

    for (const errorRecord of errors) {
      try {
        // 检查Sentry是否可用
        if (await this.isSentryAvailable()) {
          const reconstructedError = new Error(errorRecord.error.message)
          reconstructedError.stack = errorRecord.error.stack

          Sentry.captureException(reconstructedError, {
            contexts: {
              offline_sync: {
                originalTimestamp: errorRecord.metadata.timestamp,
                syncTimestamp: Date.now(),
                offlineId: errorRecord.id
              },
              ...errorRecord.context
            }
          })

          syncedIds.push(errorRecord.id)
        } else {
          break // Sentry不可用，停止同步
        }
      } catch (e) {
        console.warn('同步离线错误失败:', e)
        break
      }
    }

    // 移除已同步的错误
    if (syncedIds.length > 0) {
      const remaining = errors.filter(error => !syncedIds.includes(error.id))
      localStorage.setItem(this.storageKey, JSON.stringify(remaining))
      console.info(`✅ 已同步 ${syncedIds.length} 个离线错误`)
    }
  }

  async isSentryAvailable() {
    try {
      // 简单的连通性检查
      const testError = new Error('Connectivity Test')
      const eventId = Sentry.captureException(testError)
      return !!eventId
    } catch (e) {
      return false
    }
  }

  startSyncProcess() {
    // 页面可见时立即尝试同步
    document.addEventListener('visibilitychange', () => {
      if (!document.hidden) {
        this.syncStoredErrors()
      }
    })

    // 定期同步
    setInterval(() => {
      this.syncStoredErrors()
    }, this.syncInterval)
  }

  recoverStoredErrors() {
    // 页面加载时尝试恢复并同步离线错误
    setTimeout(() => {
      this.syncStoredErrors()
    }, 2000) // 等待2秒后开始同步
  }

  generateId() {
    return 'offline_' + Date.now() + '_' + Math.random().toString(36).substr(2, 9)
  }

  cleanupOldErrors() {
    try {
      const errors = this.getStoredErrors()
      const oneDayAgo = Date.now() - 24 * 60 * 60 * 1000

      const recentErrors = errors.filter(error =>
        error.metadata.timestamp > oneDayAgo
      )

      localStorage.setItem(this.storageKey, JSON.stringify(recentErrors))
    } catch (e) {
      // 如果还是失败，清空所有数据
      localStorage.removeItem(this.storageKey)
    }
  }
}
```

### 16.3 故障恢复流程

#### 自动恢复机制

```javascript
// utils/sentry-recovery.js
export class SentryRecoveryManager {
  constructor() {
    this.recoveryAttempts = 0
    this.maxRecoveryAttempts = 5
    this.recoveryDelay = 1000 // 初始延迟1秒
    this.isRecovering = false
  }

  async attemptRecovery() {
    if (this.isRecovering) return false

    this.isRecovering = true
    this.recoveryAttempts++

    console.info(`🔄 尝试恢复Sentry连接 (${this.recoveryAttempts}/${this.maxRecoveryAttempts})`)

    try {
      // 重新初始化Sentry
      await this.reinitializeSentry()

      // 验证连接
      if (await this.validateConnection()) {
        console.info('✅ Sentry连接已恢复')
        this.resetRecoveryState()
        return true
      }
    } catch (error) {
      console.warn('恢复失败:', error)
    }

    this.isRecovering = false

    // 如果还有重试次数，延迟后再试
    if (this.recoveryAttempts < this.maxRecoveryAttempts) {
      const delay = this.recoveryDelay * Math.pow(2, this.recoveryAttempts - 1) // 指数退避
      setTimeout(() => {
        this.attemptRecovery()
      }, delay)
    } else {
      console.error('🚨 Sentry恢复失败，已达最大重试次数')
      this.onRecoveryFailed()
    }

    return false
  }

  async reinitializeSentry() {
    // 重新导入和初始化Sentry
    const { setupSentry } = await import('./sentry')

    // 清理旧的客户端
    const client = Sentry.getCurrentHub().getClient()
    if (client) {
      await client.close()
    }

    // 重新初始化
    setupSentry()
  }

  async validateConnection() {
    try {
      const testError = new Error('Recovery Validation Test')
      const eventId = Sentry.captureException(testError)

      // 等待一小段时间确保事件被处理
      await new Promise(resolve => setTimeout(resolve, 1000))

      return !!eventId
    } catch (error) {
      return false
    }
  }

  resetRecoveryState() {
    this.recoveryAttempts = 0
    this.isRecovering = false
  }

  onRecoveryFailed() {
    // 恢复失败时的处理
    console.error('Sentry完全不可用，切换到备用监控方案')

    // 激活备用监控
    this.activateBackupMonitoring()

    // 通知开发团队
    this.notifyTeam('Sentry服务不可用，已切换到备用方案')
  }

  activateBackupMonitoring() {
    // 简单的备用错误收集
    window.addEventListener('error', (event) => {
      const errorData = {
        message: event.message,
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno,
        timestamp: new Date().toISOString(),
        userAgent: navigator.userAgent,
        url: window.location.href
      }

      // 发送到备用服务或存储到本地
      this.sendToBackupService(errorData)
    })
  }

  sendToBackupService(errorData) {
    // 发送到备用错误收集服务
    fetch('/api/backup-error-tracking', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(errorData)
    }).catch(() => {
      // 如果备用服务也不可用，存储到本地
      const backupErrors = JSON.parse(localStorage.getItem('backup_errors') || '[]')
      backupErrors.push(errorData)
      localStorage.setItem('backup_errors', JSON.stringify(backupErrors.slice(-50)))
    })
  }

  notifyTeam(message) {
    // 发送到团队通知渠道
    fetch('/api/alerts/team', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        type: 'sentry_failure',
        message,
        timestamp: new Date().toISOString(),
        severity: 'high'
      })
    }).catch(() => {
      console.error('无法发送团队通知:', message)
    })
  }
}

// 使用恢复管理器
const recoveryManager = new SentryRecoveryManager()

// 在Sentry初始化失败时触发恢复
window.addEventListener('unhandledrejection', (event) => {
  if (event.reason?.message?.includes('sentry')) {
    recoveryManager.attemptRecovery()
  }
})
```

---

**文档版本**: 2.2（完整版：技术接入 + 性能优化 + 成本控制 + 容错处理）
**更新时间**: 2025-01-21
**适用版本**: Vue 3.x + Sentry 9.x + Session Replay
