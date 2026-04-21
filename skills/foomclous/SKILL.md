---
name: foomclous
description: Use this skill when working on the FoomClous repository, especially for storage providers, Telegram Bot uploads, deployment/debugging, WebDAV behavior, large-file upload bugs, and auth/session issues.
---

# FoomClous

用于快速接手 `FoomClous` 项目二次开发。重点帮助 AI 在新对话里快速理解：

- 这是一个什么项目
- 前后端/数据库/Telegram Bot 如何协同
- 哪些文件是核心入口
- 当前已知的 WebDAV 大文件上传问题在哪里
- 为什么生产环境里会出现“Bot 上传大视频后容器重启、网页重新登录”的连锁现象

> 下面所有路径均相对仓库根目录。

## 1. 项目概述

### 用途

FoomClous 是一个自托管的个人私有云存储系统，提供：

- Web 前端文件管理界面
- Node.js 后端 API
- PostgreSQL 元数据存储
- 多存储后端切换能力
  - local
  - WebDAV
  - S3
  - 阿里云 OSS
  - OneDrive
  - Google Drive
- Telegram Bot 上传/管理入口
- 大文件分块上传、缩略图生成、视频预览、TOTP 2FA

### 技术栈

#### 后端

- Node.js + TypeScript + Express
- PostgreSQL (`pg`)
- `sharp` 处理图片缩略图
- `fluent-ffmpeg` 处理视频截图/尺寸
- `telegram` (GramJS) 做 Telegram Bot
- `webdav`, `ali-oss`, AWS S3 SDK, Google APIs

关键依赖见：

- `backend/package.json`

#### 前端

- React 19 + TypeScript
- Vite
- Tailwind CSS
- Framer Motion
- i18next

关键依赖见：

- `frontend/package.json`

### 架构

典型部署是三层：

1. `frontend/`：React + Vite 构建的静态站点
2. `backend/`：Express API + Bot + 存储抽象层
3. `postgres`：文件元数据、系统设置、账户配置

推荐理解方式：

```text
浏览器 / Telegram Bot
        ↓
   backend API / Bot
        ↓
PostgreSQL + StorageManager
        ↓
Local / WebDAV / S3 / OSS / OneDrive / Google Drive
```

## 2. 核心文件结构说明

### 仓库级

- `README.md`
  - 功能、部署方式、环境变量、反代建议
- `.env.example`
  - 运行环境变量模板
- `docker-compose.yml`
  - 默认 compose
- `docker-compose.prod.yml`
  - 生产 compose
- `init.sql`
  - PostgreSQL 初始化 SQL
- `upload_architecture.md`
  - 上传链路说明（服务器中转上传）

### 后端入口与基础设施

- `backend/src/index.ts`
  - 后端入口
  - 注册所有路由
  - 初始化 `StorageManager`
  - 初始化 Telegram Bot
  - 启动 HTTP 服务

- `backend/src/db/index.ts`
  - PostgreSQL 连接池
  - 自动初始化 schema
  - 查询封装

- `backend/src/db/schema.sql`
  - `files`、`api_keys`、`system_settings` 等表结构

### 路由层

- `backend/src/routes/auth.ts`
  - Web 登录、2FA、token 校验、登出
  - **注意：session 存在进程内存 Map 中，不持久化**

- `backend/src/routes/upload.ts`
  - 普通文件上传（小文件）

- `backend/src/routes/chunkedUpload.ts`
  - 分块上传（大文件）

- `backend/src/routes/files.ts`
  - 列表、预览、下载、删除、重命名、移动、收藏、分享

- `backend/src/routes/storage.ts`
  - 存储提供商账户配置 / 切换

### 服务层

- `backend/src/services/storage.ts`
  - **最核心的存储抽象层**
  - 各 provider 的 `saveFile / getFileStream / deleteFile / getFileSize`
  - WebDAV 问题的根源就在这里

- `backend/src/services/telegramBot.ts`
  - Telegram Bot 启动、认证、命令注册、消息分发

- `backend/src/services/telegramUpload.ts`
  - Telegram 文件下载、本地暂存、缩略图生成、调用 provider 上传、数据库写入

- `backend/src/services/orphanCleanup.ts`
  - 启动时/定时扫描 `UPLOAD_DIR`
  - 删除“不在 DB 中登记”的本地文件

- `backend/src/services/ytDlpDownload.ts`
  - `/ytdlp` 命令下载视频并入库

### 前端

- `frontend/src/App.tsx`
  - 主界面、上传队列、分类、文件夹、预览、批量操作

- `frontend/src/services/api.ts`
  - Web API 封装
  - 上传策略：小文件普通上传，大文件分块上传

- `frontend/src/services/auth.ts`
  - token 存 localStorage
  - 页面刷新时通过 `/api/auth/verify` 向后端校验

- `frontend/src/components/pages/SettingsPage.tsx`
  - 存储源管理、2FA 配置、主题、语言等

## 3. 上传与存储调用链

### Web 上传

- 普通上传入口：
  - `backend/src/routes/upload.ts:114`
- 分块上传完成后保存入口：
  - `backend/src/routes/chunkedUpload.ts:253`

### Telegram Bot 上传

Telegram 单文件上传会在下载/缩略图后调用 provider：

- `backend/src/services/telegramUpload.ts:827`
- `backend/src/services/telegramUpload.ts:1281`

也就是说，**Web 上传和 TG 上传都共用 `storage.ts` 里的 provider 实现**。  
因此 WebDAV provider 的问题会同时影响：

- 浏览器直传
- 分块上传完成后的持久化
- Telegram Bot 上传

## 4. 已知 Bug 列表

### Bug A：WebDAV 大文件上传导致内存暴涨 / 容器重启

#### 现象

- 使用 WebDAV 作为当前活动存储源
- Telegram Bot 上传大视频（在实际复现中，`>400MB` 已稳定触发）
- 文件下载到本地临时目录成功
- FFmpeg 缩略图生成成功
- 获取视频尺寸成功
- 随后 backend 进程重启
- 视频没有成功写入 WebDAV
- 本地临时文件在重启后的 orphan cleanup 中被删除
- Web 前端因为 session 丢失而重新要求登录

#### 已复现环境

- 小内存 VPS（例如 2GB RAM）
- WebDAV 作为活动 provider
- 视频文件大于约 400MB
- TG Bot 上传链路最容易触发

#### 根本原因

WebDAV provider 在上传时，把整个文件**一次性读入内存**：

- 文件：`backend/src/services/storage.ts`
- 函数：`WebDAVStorageProvider.saveFile`
- 精确位置：`317-320`

当前代码：

```ts
async saveFile(tempPath: string, fileName: string): Promise<string> {
    try {
        const fileBuffer = fs.readFileSync(tempPath);
        await this.client.putFileContents(`/${fileName}`, fileBuffer);
        console.log('[WebDAV] Upload successful:', fileName);
        return fileName;
    } catch (error: any) {
        console.error('[WebDAV] Upload failed:', error.message);
        throw new Error(`WebDAV upload failed: ${error.message}`);
    }
}
```

问题点：

- `fs.readFileSync(tempPath)` 会把整个文件读成一个大 Buffer
- 400MB 视频就会直接申请 400MB+ 的连续内存
- 在 TG Bot 场景下，前面刚经历：
  - Telegram 下载
  - FFmpeg 截图
  - sharp/webp 输出
  - PostgreSQL 查询
- 在 2GB 内存 VPS 上极易触发 Node 进程退出或容器重启

#### 关联证据链

1. 图片上传成功时，日志会出现：
   - `[WebDAV] Upload successful: xxx.jpg`
2. 大视频失败时，日志通常停在：
   - 缩略图成功
   - 视频尺寸成功
3. 然后出现完整启动日志重复：
   - `🤖 Telegram Bot 正在启动...`
   - `🚀 FoomClous 后端服务已启动`
4. 紧接着 orphan cleanup 删除临时 mp4

#### 受影响函数 / 位置

- 根因函数：
  - `backend/src/services/storage.ts:317-320`
- 触发调用点：
  - `backend/src/services/telegramUpload.ts:827`
  - `backend/src/services/telegramUpload.ts:1281`
  - `backend/src/routes/upload.ts:114`
  - `backend/src/routes/chunkedUpload.ts:253`

#### 修复思路

必须改成**流式上传**或**分块上传**，避免整文件进内存。

优先方案：

1. 用 `fs.createReadStream(tempPath)` 替代 `fs.readFileSync(tempPath)`
2. 让 WebDAV 客户端直接消费 stream
3. 如果 `webdav` 库本身对超大文件 stream 支持不足，则自行：
   - 查库的流式 API
   - 或切换到底层 HTTP PUT 流式发送
4. 确保上传成功后再删除本地临时文件
5. 对大文件增加更清晰的日志（开始上传、上传完成、上传失败）

推荐重构目标：

```text
download to temp file
 -> generate thumbnail
 -> stream upload to WebDAV
 -> insert DB row
 -> delete temp file
```

不要再出现：

```text
temp file -> read whole file into memory -> upload
```

---

### Bug B：Web 登录“7 天免登录”在容器重启后失效

#### 根因

Web session 只存在内存 Map 中：

- 文件：`backend/src/routes/auth.ts`
- 位置：`48-59`

```ts
const sessions = new Map<string, { createdAt: Date; expiresAt: Date }>();
```

token 校验依赖这个 Map：

- `backend/src/routes/auth.ts:210-224`

所以只要 backend 重启：

- session 全部丢失
- 前端 localStorage 里虽然还有 token
- 但 `/api/auth/verify` 会失败
- 页面刷新或新标签页打开都要重新登录

#### 修复思路

二选一：

1. 使用 Redis / PostgreSQL 持久化 session
2. 改成 JWT / 无状态 token 方案

如果只是快速修复，推荐先做：

- PostgreSQL session 表
- login 时写表
- verify 时查表
- logout 时删表

---

### Bug C：orphan cleanup 会删除上传失败后遗留的大文件临时文件

#### 位置

- 文件：`backend/src/services/orphanCleanup.ts`
- 核心逻辑：`97-127`

当前逻辑：

1. 读数据库中的 `stored_name`
2. 扫描 `UPLOAD_DIR`
3. 任何磁盘文件名不在 DB 里就直接删除

这在“上传流程中途崩溃”时会形成副作用：

- WebDAV 上传失败 / backend 重启
- 临时大文件仍留在 `UPLOAD_DIR`
- 但数据库还没插入记录
- orphan cleanup 把它当垃圾删掉

#### 修复思路

更合理的方式：

1. TG / Web 上传临时文件放独立 temp 目录
2. orphan cleanup 不直接扫正式 `UPLOAD_DIR` 里的一切临时文件
3. temp 文件基于：
   - 创建时间
   - 命名规则
   - 状态表
   做“延迟清理”

## 5. 开发注意事项

### 运行与调试

推荐开发模式：

- 前端本地运行：`frontend/npm run dev`
- 后端本地运行：`backend/npm run dev`
- PostgreSQL 单独运行（本地或 Docker）

原因：

- Vite / tsx watch 调试效率高
- 不必每次改代码都重建镜像

### Docker / 部署注意

- 前端 `VITE_API_URL` 是**构建时变量**
- 改前端 API 地址通常需要重新构建 frontend 镜像
- 后端 `.env` 改动通常只需重建/重建容器，不必重新 build 镜像

### 反向代理

更稳妥的生产做法：

- 只暴露 80/443 给外部
- frontend / backend 绑定 `127.0.0.1` 或只在 Docker 内网可见

### TG Bot 密码输入

README 写的是“只支持四位数字”，但当前代码更准确地说是：

- 只支持数字键盘输入
- 从第 4 位开始尝试校验
- 实际上可使用 **4 位以上纯数字密码**

如果要使用 Bot，建议：

- 使用 8~10 位纯数字密码
- 同时开启 TOTP 2FA

### 测试 WebDAV 大文件 Bug 的建议方法

复现时建议按以下顺序：

1. 将当前活动 provider 切换到 WebDAV
2. 用 TG Bot 发送一个 `>400MB` 视频
3. 观察 backend 日志：
   - 下载开始
   - 缩略图成功
   - 获取尺寸成功
   - 是否紧接着出现重复启动日志
4. 检查：
   - WebDAV 端是否无文件
   - 本地 `/data/uploads` 是否留下临时文件
   - orphan cleanup 是否删除该文件

### 修复后的验收标准

WebDAV 大文件修复后，至少要验证：

1. 400MB 视频 TG 上传成功写入 WebDAV
2. backend 进程不重启
3. DB `files` 正常插入记录
4. 本地临时文件上传完成后被正确清理
5. orphan cleanup 不误删仍在上传中的文件

## 6. 后续开发计划（建议作为 backlog）

下面是适合继续推进的修复清单：

### P0

- [ ] 修复 `WebDAVStorageProvider.saveFile()` 的整文件读内存问题，改为流式上传
- [ ] 为 TG 上传和普通上传增加更明确的“上传开始/结束/失败”日志

### P1

- [ ] 将 Web session 从内存 Map 改为 PostgreSQL/Redis/JWT 持久化方案
- [ ] 将上传临时文件从正式 `UPLOAD_DIR` 拆分到独立 temp 目录
- [ ] 调整 orphan cleanup 逻辑，避免误删未完成上传的大文件

### P2

- [ ] 为 WebDAV / TG 大文件上传增加集成测试或最小复现脚本
- [ ] 统一 README 与真实代码行为（例如 TG 密码输入限制）
- [ ] 检查 `StorageManager` 初始化与 DB 自动初始化是否存在重复启动日志噪音

## 7. 当 AI 接手这个项目时应优先做什么

如果用户提到以下关键词，应优先读取并检查这些文件：

- **WebDAV / 大文件上传 / 内存暴涨 / 容器重启**
  - `backend/src/services/storage.ts`
  - `backend/src/services/telegramUpload.ts`
  - `backend/src/services/orphanCleanup.ts`

- **刷新后重新登录 / 7 天免登录失效**
  - `backend/src/routes/auth.ts`
  - `frontend/src/services/auth.ts`

- **分块上传 / 普通上传**
  - `backend/src/routes/chunkedUpload.ts`
  - `backend/src/routes/upload.ts`
  - `frontend/src/services/api.ts`

- **存储账户配置 / 当前 provider 为什么切到 WebDAV**
  - `backend/src/routes/storage.ts`
  - `backend/src/services/storage.ts`

## 8. 一句话总结

FoomClous 是一个带 Web 前端、Node.js 后端、PostgreSQL 元数据、多存储抽象层和 Telegram Bot 的自托管网盘项目。当前最关键的生产问题是 **WebDAV provider 在上传大文件时使用 `fs.readFileSync()` 把整个文件读入内存，导致 2GB VPS 上 TG 上传 400MB+ 视频时 backend 重启，随后触发 orphan cleanup 删除临时文件，并连带造成 Web 登录 session 丢失。**
