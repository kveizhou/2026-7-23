# 2026-7-23
2026-7-23创建的第一个测试仓库
可以。你这套教程的正确链路是：
Claude Code → 本机 CodeAgent LLM Adapter（127.0.0.1:8080）→ 公司模型 API
所以，OpenCode 已经配置好并不代表 Claude Code 可以直接复用；Claude Code 要使用 Anthropic /v1/messages 协议，需要先运行教程里的 Adapter。
以下按 Windows 配置。
1. 先启动 CodeAgent LLM Adapter
进入公司教程提供的 codeagent-llm-adapter-v2 目录，安装依赖：
pip install pyyaml requests
Copy-Item config.yaml.example config.yaml
编辑 config.yaml：
server:
  host: 127.0.0.1
  port: 8080
  verbose: true

codeagent:
  zone: yellow
  insecure: true
  stream: true
  username: ${CODEAGENT_USERNAME}
  password: ${CODEAGENT_PASSWORD}

default_provider: codeagent
其中：
zone 默认使用 yellow；公司明确要求时再改成 green
insecure: true 是教程给出的内网证书兼容配置
用户名、密码不要直接写进 YAML
在 PowerShell 中临时设置公司账号：
$env:CODEAGENT_USERNAME="你的公司账号"
$env:CODEAGENT_PASSWORD="你的公司密码"
启动 Adapter：
python -m codeagent_llm_adapter --config config.yaml
这个窗口需要一直保持运行。
2. 验证 Adapter
另开一个 PowerShell：
curl.exe http://127.0.0.1:8080/v1/models
正常情况下会返回模型列表，其中至少应出现：
codeagent-default
如果这里不成功，先不要启动 Claude Code，因为问题还在 Adapter、公司网络、账号密码或区域配置这一层。
也可以直接验证 Anthropic 端点：
curl.exe http://127.0.0.1:8080/v1/messages `
  -H "Content-Type: application/json" `
  -H "x-api-key: any-string" `
  -H "anthropic-version: 2023-06-01" `
  -d '{"model":"codeagent-default","max_tokens":128,"messages":[{"role":"user","content":"回复 OK"}]}'
3. 配置 Claude Code
创建或编辑：
C:\Users\你的用户名\.claude\settings.json
建议使用下面这份配置：
{
  "env": {
    "ANTHROPIC_API_KEY": "any-string",
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8080",
    "ANTHROPIC_MODEL": "codeagent-default",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "codeagent-default",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "codeagent-default",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "codeagent-default"
  },
  "model": "codeagent-default"
}
关键点：
ANTHROPIC_BASE_URL 不要加 /v1，Claude Code 会自动请求 /v1/messages
ANTHROPIC_API_KEY 根据教程填写任意非空字符串即可；真正的公司认证由 Adapter 使用账号密码完成
后面三个 ANTHROPIC_DEFAULT_*_MODEL 可以避免 Claude Code 的子任务或快速任务改用不存在的官方 Claude 模型
不建议照截图添加 "skipDangerousModePermissionPrompt": true，这会削弱命令执行保护，和模型接入无关
Claude Code 官方支持通过 ANTHROPIC_BASE_URL 接入 LLM Gateway，也支持通过环境变量或 settings 文件设置模型。LLM Gateway 官方说明、模型配置说明
4. 启动 Claude Code
确认 Adapter 仍然运行，然后进入代码工程：
cd C:\你的项目目录
claude
第一次打开如果出现 Anthropic 登录界面，通常不要走官网登录流程；确认 settings.json 路径和 JSON 格式正确，然后完全退出 Claude Code，重新打开终端再运行。
进入 Claude Code 后可以执行：
/model
或者直接测试：
claude -p "只回复：公司模型连接成功" --model codeagent-default
如需查看详细错误：
claude --model codeagent-default --verbose
常见错误
ECONNREFUSED 127.0.0.1:8080
Adapter 没启动，或者端口不是 8080。

model not found
执行 curl.exe http://127.0.0.1:8080/v1/models，以返回的真实模型 ID 为准。

401/403
检查 Adapter 终端里的公司账号密码、区域和登录日志，不是修改 any-string。

SSL 或公司证书错误
先确认 config.yaml 中 codeagent.insecure: true。更规范的长期方案是配置公司的 CA 证书，而不是关闭证书验证。

请求仍然发往 api.anthropic.com
通常是 settings.json 放错位置、JSON 有语法错误，或 Claude Code 启动前没有重新加载配置。

OpenCode 能用但 Claude Code 不能用
OpenCode 使用的是 OpenAI 兼容接口；Claude Code 使用 Anthropic Messages 接口。重点检查 Adapter 的 /v1/messages，不要拿 /v1/chat/completions 判断 Claude Code 是否可用。
