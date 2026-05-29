# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> 默认使用简体中文回答，除非用户明确要求其他语言。

## 项目概述

JeecgBoot Vue3 前端 — 企业级低代码平台，基于 Vue 3 + Vite 6 + Ant Design Vue 4 + TypeScript 构建。使用 pnpm 作为包管理器。要求 Node 18 或 20+（`engines: "^18 || >=20"`）。

## 常用命令

```bash
pnpm dev              # 启动开发服务器（端口 3100，mock 启用）
pnpm build            # 生产构建（输出 dist/）
pnpm build:docker     # Docker 生产构建
pnpm build:dockercloud # Docker 云环境生产构建
pnpm build:report     # 构建并生成打包分析报告
pnpm preview          # 构建 + 预览

# Lint（无统一 "lint" 命令，需单独执行）
npx eslint src/path/to/file.vue          # 检查指定文件
npx stylelint "src/**/*.{vue,less,css}"  # 样式检查
pnpm batch:prettier                       # 格式化所有 src 文件

# 测试（已配置 Jest 但未集成到 npm scripts）
# 测试文件在 tests/ 目录，但 package.json 中无 test 脚本
# 如需手动运行：npx jest

pnpm clean:cache      # 清除 Vite 缓存
pnpm gen:icon         # 重新生成图标数据
pnpm reinstall        # 清理并重装所有依赖
```

## 路径别名

- `/@/` 和 `@/` → `src/`
- `/#/` 和 `#/` → `types/`
- `~icons/{collection}/{name}` → unplugin-icons（编译时图标导入）

`/@/` 前缀（带前导斜杠）是项目约定别名，为保持一致性建议优先使用。

## 架构

### 启动流程（src/main.ts）

`createApp` → createRouter → setupStore（pinia）→ setupProps → i18n → initAppConfigStore → registerPackages（@jeecg/online）→ registerGlobComp（核心 Ant Design 组件）→ SSO 登录 → registerSuper（动态模块发现）→ setupRouter → guards → directives → error handler → registerThirdComp（vxe-table、emoji、dayjs）→ setupElectron → router.isReady() → mount

### 路由与权限

- **权限模式：BACK** — 路由和菜单通过 `getBackMenuAndPerms()` 从后端 API 获取
- 动态路由在 `src/store/modules/permission.ts` 中运行时添加
- 静态路由：login、oauth2-login、token-login、error 页面、AI 仪表盘
- 路由模式：HTML5 history（Electron 环境下使用 hash 模式）
- 超级模块通过 `src/views/super/registerSuper.ts` 中的 `import.meta.glob('./**/register.ts')` 动态发现

### 状态管理（Pinia）

`src/store/modules/` 中的核心 store：

- `user.ts`（app-user）— 认证 token、用户信息、角色、租户、字典项
- `permission.ts`（app-permission）— 动态路由、权限码、后端菜单
- `app.ts`（app）— 项目配置、主题、布局设置
- `locale.ts`（app-locale）— 国际化语言
- `multipleTab.ts`（app-multiple-tab）— 多标签页状态

认证信息通过 `src/utils/auth/index.ts` 持久化到 localStorage。

### API 层

- 自定义 Axios 封装：`src/utils/http/axios/` — 配置实例导出为 `defHttp`
- 所有请求通过 `signMd5Utils` 进行 MD5 签名
- 当 `VITE_GLOB_TENANT_MODE` 启用时，租户 ID 作为请求头注入
- 响应格式：`{ code, result, message, success }`，`code === 200` 表示成功

### 组件注册

- **自动导入**：`unplugin-vue-components` 配合 `AntDesignVueResolver` 自动导入所有 Ant Design Vue 组件（模板中无需手动引入）
- **全局手动注册**：`registerGlobComp.ts` 注册 Icon、AIcon、JUploadButton、Button、TinyMCE Editor
- **第三方注册**：`registerThirdComp.ts` 注册 vxe-table（全量导入）、自定义 vxe 单元格组件、emoji picker、dayjs 插件
- **异步加载**：重型组件使用 `src/utils/factory/createAsyncComponent.tsx`

### 图标系统

三种图标方案：

1. **Iconify 运行时** — `<Icon icon="mdi:home" />`，通过 `@iconify/iconify` CDN 懒加载
2. **SVG sprites** — `<Icon icon="icon-name|svg" />`，通过 `vite-plugin-svg-icons`
3. **unplugin-icons** — `import IconName from '~icons/collection/name'`，编译时 tree-shaking

### 主题系统

- Less 变量由 `build/generate/generateModifyVars.ts` 生成
- 暗色模式通过 Ant Design Vue `theme.darkAlgorithm` 实现
- CSS 变量 `--j-global-primary-color` 由主题色动态设置
- CSS 类前缀：`jeecg`（定义在 `src/settings/designSetting.ts`）

### 外部包

- `@jeecg/online` 和 `@jeecg/aiflow` 是外部 monorepo 包，从 Vite optimizeDeps 中排除（CJS 兼容性问题）
- 通过 main.ts 中的 `registerPackages(app)` 注册

### 性能优化模式

**关键：非核心模块使用动态导入**

- 文件顶部的静态 `import` 会导致整个依赖链在初始页面加载
- 使用 `await import('module')` 或 `import('path/to/module').then()` 实现懒加载
- 使用动态导入的关键文件：
  - `src/settings/registerThirdComp.ts` — vxe-table、emoji picker（挂载后加载）
  - `src/views/super/registerSuper.ts` — 动态模块发现
  - 非关键的 Ant Design Vue 组件异步加载

**Vite optimizeDeps**

- `vite.config.ts` 中预构建的依赖包括：dayjs、axios、pinia、nprogress、qs、crypto-js、md5、sortablejs、xe-utils、vue-i18n、lodash-es、xss、mockjs
- 外部包（`@jeecg/*`）因 CJS 问题被排除

### 微前端（Qiankun）

- 可作为主应用（承载子应用）或子应用（嵌入父应用）
- 配置在 `src/qiankun/`，子应用通过 `VITE_APP_SUB_*` 环境变量配置
- 子应用模式在设置 `VITE_GLOB_QIANKUN_MICRO_APP_NAME` 时激活

### Electron 支持

- `src/electron/` — 使用 hash 路由模式
- 通过 `VITE_GLOB_RUN_PLATFORM === 'electron'` 检测平台

## 关键配置

### 环境变量

- `.env` — 基础配置（端口 3100、应用标题、SSO/qiankun 开关）
- `.env.development` — mock 启用，代理到 `localhost:8080/jeecg-boot`
- `.env.production` — mock 禁用，gzip 压缩
- `.env.docker` — Docker 生产构建配置
- `.env.dockercloud` — Docker 云环境生产构建配置
- `.env.prod_electron` — Electron 生产构建配置
- `VITE_GLOB_*` 变量通过 `dist/_app.config.js` 在运行时注入（构建后可修改）

### 构建

- 手动分包：`vue-vendor`、`antd-vue-vendor`、`vxe-table-vendor`、`emoji-mart-vue-fast`、`china-area-data-vendor`
- 构建后处理：`build/script/postBuild.ts` 生成运行时配置；`copyChat.ts` 复制聊天资源
- 生产环境通过 esbuild 移除 console/debugger

## 代码风格

- **Prettier**：150 字符宽度、单引号、尾逗号（es5）、2 空格缩进、`endOfLine: 'auto'`、`vueIndentScriptAndStyle: true`（`<script>`/`<style>` 内缩进）、`htmlWhitespaceSensitivity: 'strict'`
- **ESLint**：Vue3 recommended + TypeScript recommended + Prettier。允许 `any`。以 `_` 前缀的未使用变量被忽略。注意：`prettier/prettier` 规则设为 `'off'` — Prettier 不通过 ESLint 强制执行，需单独运行
- **提交规范**：通过 commitlint 强制执行 Conventional Commits。类型：feat、fix、perf、style、docs、test、refactor、build、ci、chore、revert、wip、workflow、types、release。Header 最大长度：108 字符
- **国际化**：支持中文（zh-CN）和英文。语言文件在 `src/locales/lang/`

## 关键目录

```
build/                    # Vite 插件、构建脚本、主题生成
src/api/                  # API 定义（sys/、common/、demo/）
src/components/jeecg/     # Jeecg 专用组件（JVxeTable、OnLine 等）
src/layouts/default/      # 主应用布局（header、sider、tabs、menu）
src/settings/             # 项目设置（design、components、locale、encryption）
src/utils/http/axios/     # HTTP 客户端配置
src/views/system/         # 系统管理页面（user、role、menu、dict 等）
src/views/super/          # 动态发现的扩展模块
types/                    # 全局 TypeScript 声明
```
