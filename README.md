# Hermes 接入中转站教程：Hermes Agent 走 Claude 中转，跑 claude-opus-5-5 全流程

Hermes Agent 本体是开源的，跑起来之后默认连官方 Anthropic 端点。国内用的时候卡在这一步：`api.anthropic.com` 连不上，Agent 起不来或者一调就超时。

这个仓库记的是在 Hermes Agent 里把上游换到 Token173（https://token173.net）的完整流程，走 Anthropic 协议，模型用 `claude-opus-5-5` 这一档。

## 准备

- 一个 Token173 账号，后台建好令牌（`sk-` 开头）
- Hermes Agent 装好，版本 0.9 以上
- 确认本机到 `token173.net` 的连通性：`curl -I https://token173.net`

## 配置

Hermes 读环境变量，两行就够：

```bash
export ANTHROPIC_BASE_URL="https://token173.net"
export ANTHROPIC_API_KEY="sk-你的令牌"
```

要固定下来写进 shell 配置：

```bash
cat >> ~/.zshrc <<'EOF'
export ANTHROPIC_BASE_URL="https://token173.net"
export ANTHROPIC_API_KEY="sk-你的令牌"
EOF
source ~/.zshrc
```

如果你用的是 `config.toml` 形式的配置，改这两项：

```toml
[anthropic]
base_url = "https://token173.net"
api_key = "sk-你的令牌"
default_model = "claude-opus-5-5"
```

字段名在不同版本里略有差异，认准 `base_url` 和 `api_key` 两个关键字即可。

## 验证是否真的走了中转

启动后先发一个最小任务，然后看两处：

1. Hermes 日志里请求的 host 是不是 `token173.net`
2. Token173 后台的调用日志里有没有对应记录

两边都对上，说明链路正常。只改了环境变量但日志里还是 `anthropic.com`，说明配置没被读到，检查是不是开在了另一个 shell 里。

## 模型选择

| 模型 ID | 适合的活儿 |
|---|---|
| `claude-opus-5-5` | 复杂重构、跨文件改代码 |
| `claude-sonnet-5-5` | 日常主力，快且稳 |
| `claude-haiku-5-5` | 批量小任务、分类抽取 |
| `claude-fable-5`、`claude-fable-5-1` | 文案、创意类任务 |
| `claude-opus-4-8` | 老配置沿用，不用改模型名 |

想让 Agent 便宜点跑，把默认模型设成 `claude-sonnet-5-5`，只在大任务时切 Opus。

## 常见问题

**Q：启动报 401？**
Key 复制不全或者前后带空格。用 `echo $ANTHROPIC_API_KEY | wc -c` 数一下长度，正常是 40 多位。

**Q：Base URL 要不要带 `/v1`？**
不要。Hermes 走 Anthropic 协议，`/v1` 是 OpenAI 那边的事，加了会 404。

**Q：流式输出卡住不动？**
先确认网络没断，再看是不是客户端版本太老不支持 SSE 重连。换最新版试一次。

**Q：能同时配两个上游吗？**
可以配多个 provider，但默认只用 `default_model` 对应的那个。多配的目的是切换方便，不是同时发。

**Q：跑长任务耗额度快怎么办？**
把上下文裁一下，别把整个仓库都喂进去。用 `claude-sonnet-5-5` 做探索，定稿再上 Opus。

## 计费

2 元 = 1 美元额度，Hermes 跑一整天的轻度使用大概几毛到一两块人民币，具体看你的任务量。后台能按令牌分开统计，方便和别的项目区分。

## 相关

- 文档站：https://docs.token173.net
- Claude 全系接入：[claude-api](https://github.com/Token173/claude-api)
- OpenClaw 配置：[openclaw-proxy](https://github.com/Token173/openclaw-proxy)
- 多模型聚合：[2026-Token173-Claude-ChatGPT-Grok-Gemini-AI-API](https://github.com/Token173/2026-Token173-Claude-ChatGPT-Grok-Gemini-AI-API)
