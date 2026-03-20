# 管理员中间件设计规范

**文档编号：** 2026-03-20-admin-middleware-design  
**作者：** AI 设计助手  
**日期：** 2026-03-20  
**状态：** 草案  
**版本：** 1.0

---

## 1. 概述

### 1.1 目的

本文档规定了**管理员中间件**的设计方案——一个面向高校教务处管理员的管理后台。该系统将爬虫管理、文件监控、通知推送和管理员多用户控制整合到基于现有 Sirchmunk Web 平台的统一管理界面中。

### 1.2 背景

高校教务处定期发布通知、政策文件和公告（如选课通知、放假安排、学生档案管理文件等）。目前，师生需手动访问教务处网站被动获取信息，导致信息滞后甚至遗漏。

Sirchmunk 平台已提供：

- **聊天界面**（`/knowledge`）用于知识问答
- **监控页面**（`/monitor`）用于系统健康和指标监控
- **设置页面**（`/settings`）用于配置管理

目前缺少的是专用的**管理员中间件**，使管理员能够：

1. 控制爬虫行为（启动/停止/配置）
2. 监控和管理已爬取的文件
3. 配置和发送通知（邮件 + 短信占位符）
4. 管理多个管理员账户

### 1.3 范围

本设计涵盖前端管理后台及其与现有后端 API 的集成。爬虫服务作为独立的 Python 服务由外部提供。

## 2. 系统架构

### 2.1 路由结构

```
web/app/
├── (admin)/                    # 管理员路由组
│   ├── layout.tsx               # 管理员专用布局（侧边栏不同）
│   ├── page.tsx                # 仪表盘概览
│   ├── crawlers/
│   │   └── page.tsx            # 爬虫管理
│   ├── tasks/
│   │   └── page.tsx            # 任务队列
│   ├── sources/
│   │   └── page.tsx            # 爬取源配置
│   ├── files/
│   │   └── page.tsx            # 文件管理与版本对比
│   ├── notifications/
│   │   └── page.tsx            # 通知配置
│   └── users/
│       └── page.tsx            # 用户管理
```

### 2.2 数据层集成

管理员中间件与主应用共享现有 API 层（`web/lib/api.ts`）：

- **API 基础地址：** 使用现有 `apiUrl()` 函数
- **身份验证：** 基于管理员角色的访问控制（待实现）
- **文件同步：** 爬取的文件自动同步到知识库
- **通知队列：** 通过 API 触发，由后端分发

### 2.3 组件架构

```
components/
├── admin/
│   ├── AdminSidebar.tsx        # 专用管理导航侧边栏
│   ├── StatCard.tsx            # 指标展示卡片
│   ├── DataTable.tsx           # 可排序/可筛选/可分页表格
│   ├── StatusBadge.tsx         # 状态指示器
│   ├── FilePreview.tsx         # 文件预览与差异对比
│   ├── Timeline.tsx            # 活动时间线
│   ├── ConfigForm.tsx           # 配置表单面板
│   └── QuickActions.tsx        # 快捷操作按钮
```

---

## 3. 页面规格

### 3.1 仪表盘（`/admin`）

**用途：** 中央概览，展示关键指标和快捷操作。

**布局：**

```
┌─────────────────────────────────────────────────────────┐
│ Header: Logo + 管理员头像 + 通知铃铛                    │
├────────┬────────────────────────────────────────────────┤
│        │ 欢迎横幅 + 快速统计行                           │
│  管理员│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────┐ │
│ 侧边栏 │ │文件总数  │ │活跃爬虫  │ │通知数量  │ │系统 │ │
│        │ │  :156   │ │  :3     │ │  :24    │ │98% │ │
│ 爬虫   │ └──────────┘ └──────────┘ └──────────┘ └────┘ │
│ 任务   │                                                  │
│ 来源   │ 近期文件表格      │  近期活动时间线              │
│ 文件   │ ─────────────────  │  ───────────────────────   │
│ 通知   │ 文件|类型|状态     │  10:30 爬虫已启动          │
│ 用户   │ ─────────────────  │  10:25 检测到新文件        │
│        │                      │  10:20 通知已发送         │
│ ────── │ 快捷操作面板       │                             │
│ 设置   │ [启动爬虫]         │                             │
│        │ [查看文件]         │                             │
│        │ [发送通知]         │                             │
└────────┴────────────────────────────────────────────────┘
```

**组件：**

- `StatCard` x 4（文件总数、活跃爬虫、已发送通知、系统健康）
- `DataTable`（近期文件，10 行）
- `Timeline`（近期活动，10 条）
- `QuickActions`（3 个按钮）

### 3.2 爬虫管理（`/admin/crawlers`）

**用途：** 监控和控制爬虫状态及配置。

**功能：**

- 爬虫开关
- 实时状态显示（运行中/空闲/错误）
- 爬取速度配置
- 调度配置（间隔式或 cron 表达式）
- 错误日志显示
- 手动触发按钮

**UI 组件：**

- 带大图标的状态指示卡片
- 启用/禁用的开关切换
- 调度配置表单
- 错误日志查看器（可滚动）

### 3.3 任务队列（`/admin/tasks`）

**用途：** 管理爬取任务队列和优先级。

**功能：**

- 任务列表：任务 ID、目标 URL、状态、进度、优先级、操作
- 状态类型：待处理、运行中、已完成、失败、已暂停
- 优先级：低、正常、高、紧急
- 任务操作：暂停、恢复、取消、重试
- 批量操作：多选 → 批量取消/重试
- 定时任务配置

**UI 组件：**

- 带状态筛选和搜索的 `DataTable`
- 每个运行中任务的进度条
- 优先级选择下拉框
- 分页（每页 20 条）

### 3.4 来源管理（`/admin/sources`）

**用途：** 配置爬取目标网站和 URL 模式。

**功能：**

- 来源网站列表：名称、URL、状态、上次爬取时间
- 添加/编辑/删除来源网站
- URL 模式配置（包含/排除规则）
- 爬取深度配置
- 文件类型过滤（HTML、PDF、DOC 等）
- 每个来源的限速配置

**UI 组件：**

- 来源列表表格
- 添加/编辑来源弹窗
- 带正则支持的 URL 模式输入框
- 文件类型复选框

### 3.5 文件管理（`/admin/files`）

**用途：** 浏览、分类、预览和对比已爬取的文件。

**功能：**

- 文件列表：名称、类型、分类、大小、状态、日期
- 分类：选课通知、放假公告、学籍管理、其他
- 文件预览（PDF、图片、文本）
- 版本对比（文本文件的 diff 视图）
- 变更历史时间线
- 文件下载
- 文件删除

**UI 组件：**

- 带分类筛选标签的 `DataTable`
- 带差异对比的 `FilePreview`
- 变更历史的 `Timeline`
- 分类徽章标签

### 3.6 通知管理（`/admin/notifications`）

**用途：** 配置通知渠道并管理通知历史。

**功能（邮件 - 第一阶段）：**

- 邮件服务器配置（SMTP 主机、端口、凭据）
- 邮件模板管理
- 测试邮件发送
- 通知历史日志

**功能（短信 - 第二阶段占位符）：**

- 短信服务商配置 UI（已禁用，待未来实现）
- API 密钥输入字段
- 短信余额显示（占位符）

**UI 组件：**

- 邮件设置配置表单
- 模板编辑器
- 通知历史表格
- 测试发送按钮

### 3.7 用户管理（`/admin/users`）

**用途：** 管理员账户管理和登录历史查看。

**功能：**

- 管理员账户列表：用户名、角色、上次登录、状态
- 添加/编辑/删除管理员账户
- 登录历史日志：时间戳、IP、结果
- 密码修改
- 会话管理（活跃会话列表）

**UI 组件：**

- 管理员账户表格
- 添加/编辑管理员弹窗
- 登录历史时间线
- 可登出的会话列表

## 4. API 集成

### 4.1 现有 API（待使用）

| 端点 | 用途 |
|------|------|
| `GET /api/v1/monitor/overview` | 系统健康指标 |
| `GET /api/v1/knowledge/stats` | 知识簇统计 |
| `GET /api/v1/settings` | 设置配置 |

### 4.2 新增管理员 API（待后端实现）

| 端点 | 方法 | 用途 |
|------|------|------|
| `/api/v1/admin/crawlers` | GET | 获取爬虫状态列表 |
| `/api/v1/admin/crawlers/toggle` | POST | 启用/禁用爬虫 |
| `/api/v1/admin/tasks` | GET/POST | 获取/创建任务 |
| `/api/v1/admin/tasks/:id` | PUT/DELETE | 更新/删除任务 |
| `/api/v1/admin/sources` | GET/POST | 获取/创建来源 |
| `/api/v1/admin/files` | GET | 获取爬取文件列表 |
| `/api/v1/admin/files/:id` | GET/DELETE | 获取/删除文件 |
| `/api/v1/admin/files/:id/preview` | GET | 获取文件预览 |
| `/api/v1/admin/files/:id/diff` | GET | 对比文件版本 |
| `/api/v1/admin/notifications` | GET/POST | 获取/创建通知 |
| `/api/v1/admin/notifications/email/config` | GET/PUT | 邮件配置 |
| `/api/v1/admin/notifications/test` | POST | 发送测试通知 |
| `/api/v1/admin/users` | GET/POST | 获取/创建管理员用户 |
| `/api/v1/admin/users/:id` | PUT/DELETE | 更新/删除用户 |
| `/api/v1/admin/login-history` | GET | 获取登录历史 |

---

## 5. 数据模型

### 5.1 爬虫状态（CrawlerStatus）

```typescript
interface CrawlerStatus {
  id: string;
  name: string;
  enabled: boolean;
  status: 'running' | 'idle' | 'error';
  lastRun: string;
  nextRun: string;
  errorMessage?: string;
}
```

### 5.2 爬取任务（CrawlTask）

```typescript
interface CrawlTask {
  id: string;
  sourceId: string;
  targetUrl: string;
  status: 'pending' | 'running' | 'completed' | 'failed' | 'paused';
  priority: 'low' | 'normal' | 'high' | 'urgent';
  progress: number;
  createdAt: string;
  completedAt?: string;
  errorMessage?: string;
}
```

### 5.3 爬取来源（CrawlSource）

```typescript
interface CrawlSource {
  id: string;
  name: string;
  baseUrl: string;
  enabled: boolean;
  includePatterns: string[];
  excludePatterns: string[];
  fileTypes: string[];
  crawlDepth: number;
  rateLimit: number;
  lastCrawl?: string;
}
```

### 5.4 爬取文件（CrawledFile）

```typescript
interface CrawledFile {
  id: string;
  name: string;
  type: 'pdf' | 'html' | 'doc' | 'docx' | 'other';
  category: 'course_selection' | 'holiday' | 'student_records' | 'other';
  size: number;
  path: string;
  status: 'new' | 'processed' | 'notified';
  version: number;
  createdAt: string;
  updatedAt: string;
}
```

### 5.5 管理员用户（AdminUser）

```typescript
interface AdminUser {
  id: string;
  username: string;
  email: string;
  role: 'admin' | 'super_admin';
  status: 'active' | 'inactive';
  lastLogin?: string;
  createdAt: string;
}
```

### 5.6 通知记录（NotificationRecord）

```typescript
interface NotificationRecord {
  id: string;
  fileId: string;
  fileName: string;
  channel: 'email' | 'sms';
  recipients: string[];
  status: 'sent' | 'failed' | 'pending';
  sentAt: string;
  errorMessage?: string;
}
```

## 6. 文件结构

```
web/
├── app/
│   ├── (admin)/              # 路由组
│   │   ├── layout.tsx
│   │   ├── page.tsx          # 仪表盘
│   │   ├── crawlers/
│   │   ├── tasks/
│   │   ├── sources/
│   │   ├── files/
│   │   ├── notifications/
│   │   └── users/
│   └── globals.css           # 扩展管理员变量
├── components/
│   └── admin/
│       ├── AdminSidebar.tsx
│       ├── StatCard.tsx
│       ├── DataTable.tsx
│       ├── StatusBadge.tsx
│       ├── FilePreview.tsx
│       ├── Timeline.tsx
│       ├── ConfigForm.tsx
│       └── QuickActions.tsx
└── lib/
    └── admin-api.ts          # 管理员专用 API 工具
```


*文档版本：1.0*  
*最后更新：2026-03-20*
