# Design: 使用在线 CPA 管理页面替代 Docker

## 已确认的 API 接口

通过逆向分析 `management.html` 并实测，在线 CPA 服务提供以下接口：

| 方法 | 路径 | 请求头 | 响应 |
|------|------|--------|------|
| GET | `/v0/management/qwen-auth-url` | `Authorization: Bearer <key>` | `{state, status, url}` |
| GET | `/v0/management/get-auth-status?state=<state>` | `Authorization: Bearer <key>` | `{status: "wait"|"ok"}` |
| GET | `/v0/management/auth-files` | `Authorization: Bearer <key>` | `{files: [{id, email, provider, ...}]}` |
| GET | `/v0/management/auth-files/download?name=<id>` | `Authorization: Bearer <key>` | 凭证 JSON 文件内容 |

实测响应示例：
```json
// GET /v0/management/qwen-auth-url
{"state":"gem-1774154837064623469","status":"ok","url":"https://chat.qwen.ai/authorize?user_code=NFEQ6O-P&client=qwen-code"}

// GET /v0/management/get-auth-status?state=gem-...
{"status":"wait"}  // 认证前
{"status":"ok"}   // 认证完成

// GET /v0/management/auth-files
{"files":[{"id":"qwen_oauth_xxx.json","email":"xxx","provider":"qwen","created_at":"..."}]}
```

## 现有架构 vs 新架构

**现有（Docker）**：
```
provision_qwen_oauth_credentials()
  ├── QwenOAuthLoginRunner.start()           # docker run eceasy/cli-proxy-api --qwen-login
  ├── QwenOAuthLoginRunner.wait_for_authorize_url()  # 解析 stdout
  ├── QwenOAuthBrowserAutomator.authorize()  # Playwright 登录
  ├── QwenOAuthLoginRunner.wait_for_identity_prompt()
  ├── QwenOAuthLoginRunner.submit_identity() # 向 stdin 写 email
  ├── QwenOAuthLoginRunner.wait_for_credentials()    # 等待文件写入
  └── QwenOAuthLoginRunner.close()           # 关闭容器
```

**新（HTTP API）**：
```
provision_qwen_oauth_credentials()
  ├── OnlineCpaClient.get_qwen_auth_url()    # GET /v0/management/qwen-auth-url
  ├── QwenOAuthBrowserAutomator.authorize()  # 不变，复用
  ├── OnlineCpaClient.poll_auth_status()     # GET /v0/management/get-auth-status?state=...
  └── OnlineCpaClient.download_credential() # GET /v0/management/auth-files + download
```

## OnlineCpaClient 类设计

```python
class OnlineCpaClient:
    def __init__(self, *, base_url: str, api_key: str, timeout: float = 20.0)

    def get_qwen_auth_url(self) -> dict
    # GET /v0/management/qwen-auth-url
    # 返回 {"state": str, "url": str}

    def poll_auth_status(self, state: str, timeout_seconds: float = 120.0) -> None
    # 轮询 GET /v0/management/get-auth-status?state=...
    # 每 2 秒一次，直到 status==ok 或超时

    def get_latest_credential_by_email(self, email: str) -> dict
    # GET /v0/management/auth-files 过滤 provider==qwen 且 email 匹配
    # 返回最新文件的完整凭证 JSON
```

## provision_qwen_oauth_credentials 修改

函数签名新增可选参数，保持向后兼容：

```python
def provision_qwen_oauth_credentials(
    *,
    email: str,
    password: str,
    output_dir: Path,
    headed: bool = True,
    # 新增：传入则走在线路径，不传则保留 Docker 路径
    cpa_client: OnlineCpaClient | None = None,
    # 保留（仅 Docker 路径使用）
    runner: QwenOAuthLoginRunner | None = None,
    browser_automator: QwenOAuthBrowserAutomator | None = None,
    work_dir: Path | None = None,
    log_fn: Callable[[str], None] | None = None,
) -> dict[str, Any]
```

## qwen_register.py 修改

`register_once()` 中构建 `oauth_provisioner` 时，检测环境变量并自动选择路径：

```python
# 若 CLI_PROXY_API_BASE_URL 和 CLI_PROXY_API_KEY 已配置，使用在线路径
cpa_client = OnlineCpaClient(
    base_url=args.cli_proxy_api_base_url,
    api_key=args.cli_proxy_api_key,
) if args.cli_proxy_api_base_url and args.cli_proxy_api_key else None

oauth_provisioner = lambda **kw: provision_qwen_oauth_credentials(
    cpa_client=cpa_client, **kw
)
```

## 凭证获取策略

认证成功后（`status=ok`），需从 `/v0/management/auth-files` 中找到本次新增的凭证：
- 按 `email` 过滤，`provider=qwen`
- 取 `created_at` 最新的文件
- 调用 `/v0/management/auth-files/download?name=<id>` 下载 JSON 内容
- 写入本地 `output_dir`

## 环境变量

复用现有变量，无需新增：

| 变量 | 用途 |
|------|------|
| `CLI_PROXY_API_BASE_URL` | 在线 CPA 服务地址（如 `http://new-api.soeasy.space:8317`）|
| `CLI_PROXY_API_KEY` | 管理密钥（如 `kevinr1990`）|
