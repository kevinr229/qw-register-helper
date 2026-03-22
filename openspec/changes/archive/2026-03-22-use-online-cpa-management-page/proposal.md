# Proposal: 使用在线 CPA 管理页面替代 Docker 本地运行

## 背景

当前 Qwen OAuth 认证流程依赖本地 Docker 运行 `eceasy/cli-proxy-api` 容器（`--qwen-login` 模式）。这要求运行环境必须安装并能启动 Docker，在轻量级服务器、纯 Python 环境或 Windows 无 WSL 的场景下无法使用。

## 已探明的在线 CPA API

在线 CPA 管理服务（如 `http://new-api.soeasy.space:8317`）已提供等效的 HTTP API：

| 步骤 | 方法 | 路径 | 说明 |
|------|------|------|---------|
| 获取授权链接 | GET | `/v0/management/qwen-auth-url` | 返回 `{state, url}` |
| 轮询认证状态 | GET | `/v0/management/get-auth-status?state=...` | 返回 `{status: "wait"/"ok"}` |
| 获取凭证文件列表 | GET | `/v0/management/auth-files` | 返回已保存的凭证文件 |

认证请求头：`Authorization: Bearer <management_key>`

## 方案

将 `qwen_oauth_login.py` 中的 `QwenOAuthLoginRunner`（Docker 容器管理）替换为 `OnlineCpaClient`（HTTP 客户端）。`QwenOAuthBrowserAutomator` 保持不变复用。

新流程：
1. `OnlineCpaClient` 调用 `/v0/management/qwen-auth-url` → 获取 `state` 和 `authorize_url`
2. `QwenOAuthBrowserAutomator` 用 Playwright 打开 `authorize_url`，用账密完成 Qwen 登录授权
3. `OnlineCpaClient` 轮询 `/v0/management/get-auth-status?state=...` 直到 `status=ok`
4. 从 `/v0/management/auth-files` 获取最新生成的凭证 JSON，保存到本地

## 目标

- 移除对 Docker 的强依赖，只需 Python + Playwright
- 保持 `provision_qwen_oauth_credentials` 函数签名向后兼容
- 通过环境变量 `CLI_PROXY_API_BASE_URL` + `CLI_PROXY_API_KEY` 配置在线服务（复用现有变量）

## 非目标

- 不修改注册、激活、token 获取等主流程
- 不修改 `CloudflareTempEmailClient` 和 `RouterManagementClient`
- 不重写 `QwenOAuthBrowserAutomator`
