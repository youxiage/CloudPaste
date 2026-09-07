# GitHub Fork 与 Workers Builds 部署设计

## 范围

将 Sink、CloudFlare-ImgBed 和 CloudPaste 分别 Fork 到 `youxiage`，保留各自的独立 Worker、域名、D1、KV 和 R2 资源，并把现有生产配置提交到对应 Fork。

## 发布流程

每个现有 Worker 连接其同名 GitHub Fork。`main` 或 `master` 作为生产分支；推送后由 Cloudflare Workers Builds 安装依赖、构建并运行 Wrangler 部署。开发和故障恢复仍可使用本地 Wrangler。

## 安全边界

仓库只保存 Worker 名称、域名、资源名称和资源 ID。管理员密码、R2 S3 密钥、`ENCRYPTION_SECRET`、Cloudflare API 令牌及其他凭据不进入 Git。现有 Worker Secret 在部署后继续保留。

## 构建命令

- Sink：安装 pnpm 依赖并执行 Nuxt 构建，然后使用 `wrangler.jsonc` 部署。
- CloudFlare-ImgBed：安装 npm 依赖并运行 `npm run deploy:worker`。
- CloudPaste：仓库根目录运行 `npm run build`，分别安装并构建前端、安装后端依赖；随后运行 `npm run deploy`。

## 验证

每个 Fork 推送后检查 GitHub 内容不含密钥，触发一次 Workers Build，确认对应自定义域名和 API 健康检查正常。CloudPaste 额外验证 WebDAV 的读写、移动、下载和删除链路。
