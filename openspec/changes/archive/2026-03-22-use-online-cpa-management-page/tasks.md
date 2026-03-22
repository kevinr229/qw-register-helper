# Tasks: 使用在线 CPA 管理页面替代 Docker

## [x] Task 1: 在 qwen_oauth_login.py 中新增 OnlineCpaClient 类

在 `qwen_oauth_login.py` 顶部（`QwenOAuthLoginRunner` 之前）添加 `OnlineCpaClient` 类，使用 `requests` 实现以下三个方法：

### `get_qwen_auth_url() -> dict`
- `GET /v0/management/qwen-auth-url`
- 请求头：`Authorization: Bearer <api_key>`
- 返回响应 JSON（含 `state` 和 `url` 字段）
- 若 HTTP 非 2xx 抛出异常

### `poll_auth_status(state: str, timeout_seconds: float = 120.0) -> None`
- 循环 `GET /v0/management/get-auth-status?state=<state>`
- 请求头：`Authorization: Bearer <api_key>`
- 每 2 秒轮询一次
- 响应 `{\"status\": \"ok\"}` 时返回
- 超时抛出 `TimeoutError`

### `get_credential_by_email(email: str, after_ts: float, timeout_seconds: float = 30.0) -> dict`
- 轮询 `GET /v0/management/auth-files`
- 过滤 `provider == \"qwen\"` 且 `email` 匹配
- 过滤 `created_at` 晚于 `after_ts`（Unix 时间戳）
- 找到后调用 `GET /v0/management/auth-files/download?name=<id>` 获取完整凭证 JSON
- 超时抛出 `TimeoutError`

```python
class OnlineCpaClient:
    def __init__(self, *, base_url: str, api_key: str, timeout: float = 20.0) -> None:
        self.base_url = base_url.rstrip(\"/\")
        self.api_key = api_key
        self.session = requests.Session()
        self.timeout = timeout

    def _headers(self) -> dict:
        return {\"Authorization\": f\"Bearer {self.api_key}\"}
```

---

## [x] Task 2: 修改 provision_qwen_oauth_credentials 支持在线路径

在 `qwen_oauth_login.py` 的 `provision_qwen_oauth_credentials` 函数中：

1. 新增可选参数 `cpa_client: OnlineCpaClient | None = None`
2. 当 `cpa_client` 不为 None 时，走在线路径：
   ```
   记录 before_ts = time.time()
   url_data = cpa_client.get_qwen_auth_url()
   authorize_url = url_data[\"url\"]
   state = url_data[\"state\"]
   browser_automator.authorize(authorize_url, email, password)
   cpa_client.poll_auth_status(state)
   credential = cpa_client.get_credential_by_email(email, after_ts=before_ts)
   将 credential 写入 output_dir / f\"qwen_oauth_{safe_email_name(email)}.json\"
   返回 {\"status\": \"success\", \"oauth_file\": ..., \"oauth_payload\": credential}
   ```
3. 当 `cpa_client` 为 None 时，保持原有 Docker 路径不变（不删除）
4. 在线路径中同样调用 `log_fn` 输出阶段日志，与 Docker 路径日志文案保持一致

---

## [x] Task 3: 修改 qwen_register.py 自动选择在线路径

在 `register_once()` 函数中：

1. 新增 `cpa_client` 参数（`OnlineCpaClient | None`），从 `main()` 传入
2. 构建 `oauth_provisioner` 时将 `cpa_client` 传给 `provision_qwen_oauth_credentials`：
   ```python
   oauth_data = provision_qwen_oauth_credentials(
       email=email,
       password=password,
       output_dir=out_dir,
       headed=oauth_headed,
       cpa_client=cpa_client,  # 新增
       log_fn=...,
   )
   ```
3. 在 `main()` 中，若 `args.cli_proxy_api_base_url` 和 `args.cli_proxy_api_key` 均存在，构建 `OnlineCpaClient`：
   ```python
   cpa_client = None
   if args.cli_proxy_api_base_url and args.cli_proxy_api_key:
       cpa_client = OnlineCpaClient(
           base_url=args.cli_proxy_api_base_url,
           api_key=args.cli_proxy_api_key,
       )
   ```
4. 将 `cpa_client` 传入 `run_registration_batch` → `register_once` 的 lambda

**注意**：`upload_client`（`RouterManagementClient`）仍独立存在，不受影响。

---

## [x] Task 4: 更新 README.md

在 README 中说明：
- 若配置了 `CLI_PROXY_API_BASE_URL` 和 `CLI_PROXY_API_KEY`，注册时将自动使用在线 CPA 管理页面完成 OAuth，**无需 Docker**
- Docker 方式仍可用（不配置上述变量时的降级路径）
- 在