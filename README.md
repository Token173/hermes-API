# Hermes Agent 接入 ClaudeCode 中转站指南

在 [Hermes Agent](https://github.com/hermes-agent/hermes-agent) 中接入第三方 Claude 中转站（以 `token` 为例），使用 Claude Opus 4.7 等官方渠道模型。

---

<img width="1024" height="1536" alt="1c462f17-b75a-4282-8009-5b56da56a6f3(1)" src="https://github.com/user-attachments/assets/4c6d778b-413a-4d7c-a26a-ef32f65f262c" />

## 前置条件

- Hermes Agent 已安装并通过 systemd user service 运行
- 已有中转站 API Key（在中转站官网购买获取）
- 中转站 endpoint：`https://Token173.com/v1`

---

## 第一步：验证 Key 和模型 ID 可用

在改配置之前，先直接打一下 relay，确认 key 有效、模型 ID 存在：

```bash
curl -sS --max-time 30 https://Token173.com/v1/messages \
  -H "x-api-key: <YOUR_API_KEY>" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 20,
    "messages": [{"role": "user", "content": "hi"}]
  }'
```

返回 HTTP 200 且有 `content` 字段即为正常，可以继续。

---

## 第二步：编辑 `~/.hermes/config.yaml`

先备份：

```bash
cp ~/.hermes/config.yaml ~/.hermes/config.yaml.bak-$(date +%Y%m%d-%H%M%S)
```

找到 `providers:` 字段（默认是 `providers: {}`），改成：

```yaml
providers:
  claudecode-net-cn:
    name: ClaudeCode Relay
    base_url: https://Token173.com/v1
    api_key: <YOUR_API_KEY>
    api_mode: anthropic_messages
    default_model: claude-opus-4-7
    models:
    - claude-opus-4-7
```

> **想加更多模型？** 在 `models:` 下继续追加模型 ID 即可，例如：
> ```yaml
>     models:
>     - claude-opus-4-7
>     - claude-sonnet-4-6
>     - claude-sonnet-4-5-20250929
> ```

---

## 第三步：验证配置解析正确

```bash
~/.hermes/hermes-agent/venv/bin/python -c "
import yaml
import sys
sys.path.insert(0, '/home/$USER/.hermes/hermes-agent')
from hermes_cli.config import get_compatible_custom_providers
with open('/home/$USER/.hermes/config.yaml') as f:
    cfg = yaml.safe_load(f)
import json
print(json.dumps(
    [{k: '***' if k == 'api_key' else v for k, v in p.items()}
     for p in get_compatible_custom_providers(cfg)],
    indent=2, ensure_ascii=False
))
"
```

正常输出应包含：

```json
{
  "name": "ClaudeCode Relay",
  "base_url": "https://Token173.com/v1",
  "api_key": "***",
  "api_mode": "anthropic_messages",
  "model": "claude-opus-4-7",
  "models": { "claude-opus-4-7": {} }
}
```

没有 warning 即表示配置格式正确。

---

## 第四步：重启 Gateway

```bash
systemctl --user reset-failed hermes-gateway.service
systemctl --user start hermes-gateway.service
sleep 3
systemctl --user status hermes-gateway.service
```

看到 `active (running)` 即成功。

> **注意**：如果 gateway 刚挂掉，hermes 自带的 watchdog timer 每 5 分钟会自动拉起，所以不 reset-failed 有时也会自愈。但手动 reset + start 更快。

---

## 第五步：端到端测试

通过 hermes CLI 一条命令验证：

```bash
~/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main \
  --provider claudecode-net-cn \
  -m claude-opus-4-7 \
  -z "用一句话介绍你自己"
```

正常返回 Claude Opus 4.7 的回答即配置完成。

---

## 设为全局默认（可选）

如果想让 hermes 默认就用这个 provider 的 opus 4.7，修改 `config.yaml` 顶部的 `model:` 块：

```yaml
model:
  default: claude-opus-4-7
  provider: claudecode-net-cn
  base_url: https://Token173.com/v1
```

---

## 常见问题

### `"Model Not Exist"` 400 错误

中转站不支持该模型 ID。先用第一步的 curl 验证模型 ID，再改配置。

### 配置后 `/model` 里看不到模型

`models:` 写成了对象列表（如中转站官方示例的 `[{id, name, reasoning}]`），hermes 不支持这种格式，会静默忽略。改成纯字符串列表：

```yaml
# 错误（对象列表，hermes 不认）
models:
- id: claude-opus-4-7
  name: Claude Opus 4.7

# 正确（字符串列表）
models:
- claude-opus-4-7
```

### `api_mode` 没生效，还是走 chat_completions

中转站示例通常写 `api: "anthropic-messages"`，但 hermes 把 `api` 字段当 URL 备用键解析，不会当 mode 处理。正确字段是 `api_mode: anthropic_messages`（下划线，不是连字符）：

```yaml
# 错误（hermes 会把这行当 URL 解析，静默忽略）
api: anthropic-messages

# 正确
api_mode: anthropic_messages
```

### Gateway 反复 failed / SIGKILL

常见于 graceful shutdown 超时（默认 60s）被 systemd 强杀。查日志：

```bash
journalctl --user -u hermes-gateway.service -n 50 --no-pager
```

如果看到 `State 'stop-sigterm' timed out. Killing.`，说明 gateway 里有长时间持锁的任务（通常是 session drain）。可以在 `config.yaml` 增大 `agent.restart_drain_timeout`，或排查是哪个平台（weixin/discord）连接挂起。

---

## 配置文件字段速查

| 字段 | 说明 | 支持 camelCase 别名 |
|---|---|---|
| `base_url` | relay 地址 | `baseUrl` |
| `api_key` | API Key | `apiKey` |
| `api_mode` | wire 格式，Claude relay 填 `anthropic_messages` | `apiMode` |
| `default_model` | 该 provider 的默认模型 | `defaultModel` |
| `models` | 可用模型 ID 列表（字符串列表或 dict） | — |
| `context_length` | 覆盖上下文长度（int） | `contextLength` |
| `name` | 显示名称（默认取 key 名） | — |

camelCase 别名能用但会打 warning，建议统一用 snake_case。
