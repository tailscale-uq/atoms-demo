# Atoms Demo · AI 应用工作台

[在线体验](https://atoms-workbench-tl-0926.tong37938.chatgpt.site/)

完整源码在仓库的 `atoms-demo-source.zip`，下载解压后执行下方启动步骤。压缩包包含全部应用源码、依赖锁文件、数据库迁移和测试；不含密钥、依赖目录或本地数据库。

已接入 Groq 的 `openai/gpt-oss-120b`，完成真实生成、多轮修改、数据库读回及版本回退的线上 API 验收。具体范围见 `verification.json`。浏览器按钮交互尚未做完整自动化回归。


一个可以运行的对话式网页应用生成器：描述需求 → 规划 → 编写 HTML → 结构检查（最多修复一次）→ 隔离预览 → 保存版本 → 继续修改。

## 功能与边界

- 真实 AI 模式：服务端调用兼容 Chat Completions 的 HTTPS 模型服务；两次调用分别规划和写代码，结构不合格再修复一次。
- 示例模式：无需密钥，明确标识为固定模板，包含任务看板、收支账本、专注时钟，不冒充 AI 生成，也不处理自定义自然语言。
- Cloudflare D1 保存项目、需求、计划、代码及全部历史版本。恢复仅切换 current 指针，不删除历史。
- HTML 源码编辑、保存新版本、HTML 下载、桌面和移动端预览。
- 随机 HttpOnly 会话 cookie 隔离不同浏览器的数据，所有项目读写均校验归属。没有账号注册与跨设备同步；会话 30 天过期或清除 cookie 后无法找回原匿名项目。
- 预览 iframe 只开 allow-scripts，禁止同源权限，CSP 限制外部资源、网络请求和表单提交。预览内的任务、账单等示例运行数据仅在内存中；工作台项目与代码在数据库持久化。
- 不支持任意 npm 包、外部 API、生成应用的独立后端或自动部署。结构检查不是完整的浏览器功能测试，也不能保证任意 AI 输出正确。
- 发布多用户生产服务前应补充账号、跨实例频率限制、每日额度与任务队列；当前 UI 防重复点击并设置 180 秒生成超时，不能替代服务端限流。

## 本地启动

需要 Node.js >=22.13 与 npm。

```sh
npm ci
npm run build
npm run db:init
npm run dev -- --host 127.0.0.1
```

本地地址以开发服务器输出为准，通常是 http://localhost:3000 。`db:init` 对本地开发数据库执行版本化迁移，可重复运行；只应在本地执行。

## 模型连接

复制 `.dev.vars.example` 为 `.dev.vars`，在本机填写以下值后重启开发服务器：

```text
LLM_API_KEY=你的模型密钥
LLM_BASE_URL=https://api.deepseek.com
LLM_MODEL=deepseek-flash
```

URL 应是 Chat Completions 服务的基地址，不包含 `/chat/completions`；选择支持 system/user 消息和 max_tokens 的兼容模型。正式部署在平台配置对应 secret / 环境变量。密钥不进入浏览器、版本记录或 Git。模型名示例来自提供商文档，实际以账号可用模型为准。

未配置时 `/api/bootstrap` 返回 `aiReady: false`，AI 按钮禁用；AI 请求返回 503，不能把示例体验当作真实 AI 验收。只有完成真实模型端到端测试后，才能声明 AI 生成已实测。

## 测试

```sh
npm test
npm run typecheck
# 开发服务器运行、数据库已初始化后：
npm run test:integration
```

单元测试使用模拟模型响应验证规划、编写、一次修复、认证失败与输出截断；不消耗真实额度。接口测试使用独立匿名会话，验证三个模板、读回、编辑、回退、跨会话拒绝、来源校验和缺失模型配置，会留下一个测试项目。集成测试预期未配置模型。

## 架构

- React 19 + Vinext/Vite + TypeScript；UI 基于已安装的 Base UI / shadcn primitives。
- Cloudflare Worker API + D1；Drizzle 负责 schema 与迁移，查询采用参数化语句。
- `lib/agent.ts`：有界规划/生成/结构修复编排。
- `app/api/generate/route.ts`：NDJSON 阶段事件、取消/超时、版本保存。
- `lib/validation.ts`：HTML 结构及外部依赖检查、预览 CSP。
- `app/api/projects`：项目归属验证、原子版本保存与回退。
- `tests`：无需额外测试框架的单元与接口测试。

NDJSON 仅流式反馈阶段与计划，代码完成后一次返回；不是 token 流式显示。中途停止后若保存已开始，应重新打开项目确认最终状态。

## 部署与提交

`.openai/hosting.json` 绑定本次 Sites 项目及 D1，平台应用 `drizzle/` 迁移。复制源码用于其他项目时，应移除原 project_id 并重新配置目标项目，不能误用原项目身份。

本次演示已公开并连接真实模型。自行部署时需要配置自己的模型密钥。请不要将招聘方原始笔试文档上传至公开仓库。当前源码不包含原始文档或个人账单。

参考：[Atoms](https://atoms.dev/)、[产品帮助中心](https://help.atoms.dev/en)、[模型接口文档](https://api-docs.deepseek.com/api/create-chat-completion/)。

## 依赖审计记录

2026-09-26 对官方模板的 `npm audit --omit=dev` 报告 5 个 high 依赖条目（包含传递依赖）。本次保留模板锁定版本以保证平台兼容，没有盲目强制升级；公开多人部署前需评估并升级兼容版本，重新执行回归。该应用未实现 Server Actions、文件上传、图片优化或代理功能，但这不构成“没有风险”的保证。
