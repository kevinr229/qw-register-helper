

> **⚠️ 声明：本仓库代码仅供学习交流使用，严禁用于任何商业用途！如果您觉得有帮助，欢迎点个 Star ⭐️ 支持一下！**

## 环境准备
1. 创建并激活虚拟环境（Python 3.9+）：
```bash
cd qwen-register
python3 -m venv venv
source venv/bin/activate
# 或者用 pycharm打开，安装好python解释器
```

2. 安装依赖：
```bash
pip install -r requirement.txt
```

3. 配置环境变量，可新建.env文件：
```bash
CLOUDFLARE_TEMP_EMAIL_BASE_URL=https://example.com/
ADMIN_PASSWORDS='["***","***"]'
CLI_PROXY_API_BASE_URL=http://example.com:8317
CLI_PROXY_API_KEY=sk-***
QWEN_REGISTER_COUNT=20
```

## 运行
```bash
cd qwen-register
source venv/bin/activate
python qwen_register.py
# 在pycharm环境中，直接运行 qwen_register.py文件
```

由于oauth认证动作依赖cpa的认证和回调，这里需要启动一个docker镜像，要求运行的环境有可以运行的docker环境。
可以是Mac Ubuntu环境，或者Windows里的 WSL环境，安装好docker。

默认会批量注册 `5` 个账号。可以通过 `--count` 或环境变量 `QWEN_REGISTER_COUNT` 覆盖：

```bash
python qwen_register.py --count 1
python qwen_register.py --count 10
```

批量模式下：
- 单条失败会记录到结果里，并继续后续注册
- 每两条注册之间会随机等待 `10-30` 秒
- 最终会输出 `success_count` 和 `failed_count` 统计
- 运行中的阶段日志输出到 `stderr`，最终 JSON 结果输出到 `stdout`

如果要在注册成功后自动注册、激活、生成官方 Qwen OAuth 凭证并上传到 CLIProxyAPI：

```bash
python qwen_register.py --count 5 \
  --cli-proxy-api-base-url http://example.com:8317 \
  --cli-proxy-api-key sk-*** \
  --oauth-headed
```
