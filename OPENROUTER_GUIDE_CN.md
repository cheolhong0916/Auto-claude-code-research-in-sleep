# OpenRouter 接入指南

本文档说明如何使用 [OpenRouter](https://openrouter.ai/) 驱动 ARIS 全流程，支持访问 200+ 种模型（包括免费模型）。

---

## 背景

### OpenRouter 是什么

[OpenRouter](https://openrouter.ai/) 是一个统一的 AI 模型 API 网关，提供：
- **200+ 模型**：OpenAI、Anthropic、Google、DeepSeek、MiniMax、Qwen 等
- **免费模型**：部分模型提供免费额度（如 `minimax/minimax-m2.5:free`）
- **统一接口**：标准 OpenAI-compatible API，一个 Key 访问所有模型
- **价格透明**：按量计费，支持加密货币支付

### 支持的免费模型（推荐）

| 模型 | 提供商 | 用途 | 说明 |
|------|--------|------|------|
| `minimax/minimax-m2.5:free` | MiniMax | 审查器 | 免费版 MiniMax-M2.5 ||

> 完整模型列表：https://openrouter.ai/models

---

## 双层架构

```
┌──────────────────────────────────────────────────────────┐
│                    Claude Code (CLI)                      │
│                                                           │
│  ┌──────────────────┐       ┌─────────────────────────┐  │
│  │     执行器        │──────▶│        审查器            │  │
│  │  (Claude CLI)    │       │   (llm-chat MCP)        │  │
│  │                  │       │                         │  │
│  │  ANTHROPIC_* 变量 │       │  LLM_* 环境变量          │  │
│  │  → Anthropic 端点 │       │  → OpenRouter 端点       │  │
│  └──────────────────┘       └─────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

| 角色 | 协议 | 端点 |
|------|------|------|
| 执行器（Claude CLI） | Anthropic 兼容 | OpenRouter Anthropic 端点 或 其他 |
| 审查器（llm-chat MCP） | OpenAI 兼容 | `https://openrouter.ai/api/v1` |

---

## 获取 API Key

1. 访问 [OpenRouter](https://openrouter.ai/) 注册账号
2. 进入 [Keys 页面](https://openrouter.ai/keys) 创建 API Key
3. 格式：`sk-or-v1-xxxxxxxxxxxxxxxx`
4. 免费模型无需充值即可使用

---

## 安装步骤

### 前置条件

- Claude Code CLI 已安装：`npm install -g @anthropic-ai/claude-code`
- Python 3 可用
- 已获取 OpenRouter API Key

### Step 1：克隆仓库

```bash
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git
cd Auto-claude-code-research-in-sleep
```

### Step 2：安装 Python 依赖

```bash
pip3 install -r mcp-servers/llm-chat/requirements.txt
```

### Step 3：部署 llm-chat MCP 服务器

```bash
mkdir -p ~/.claude/mcp-servers/llm-chat
cp mcp-servers/llm-chat/server.py ~/.claude/mcp-servers/llm-chat/server.py
```

### Step 4：安装 Skills

```bash
mkdir -p ~/.claude/skills
cp -r skills/* ~/.claude/skills/
```

### Step 5：配置 ~/.claude/settings.json

**方案 A：执行器也用 OpenRouter（Anthropic 协议）**

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "sk-or-v1-your-openrouter-key",
    "ANTHROPIC_BASE_URL": "https://openrouter.ai/api",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "openrouter/free", 
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "openrouter/free",
    "ANTHROPIC_SMALL_FAST_MODEL": "openrouter/free",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "6000"
  },
  "mcpServers": {
    "llm-chat": {
      "command": "/usr/bin/python3",
      "args": ["$HOME/.claude/mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "sk-or-v1-your-openrouter-key",
        "LLM_BASE_URL": "https://openrouter.ai/api/v1",
        "LLM_MODEL": "openrouter/free"
      }
    }
  }
}
```

**方案 B：执行器用其他 API + 审查器用 OpenRouter（推荐）**

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your-executor-api-key",
    "ANTHROPIC_BASE_URL": "https://api.anthropic.com",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-6",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "6000"
  },
  "mcpServers": {
    "llm-chat": {
      "command": "/usr/bin/python3",
      "args": ["$HOME/.claude/mcp-servers/llm-chat/server.py"],
      "env": {
        "LLM_API_KEY": "sk-or-v1-your-openrouter-key",
        "LLM_BASE_URL": "https://openrouter.ai/api/v1",
        "LLM_MODEL": "openrouter/free"
      }
    }
  }
}
```

> **路径说明**：`$HOME` 需替换为实际路径（如 `/root` 或 `/home/username`），`python3` 路径用 `which python3` 确认。

### Step 6：改写所有使用 Codex MCP 的 Skills

启动 Claude Code 后执行：

```
Read skills/auto-review-loop-llm/SKILL.md as a reference.
It replaces mcp__codex__codex with mcp__llm-chat__chat.
Now rewrite ALL other skills that use mcp__codex__codex / mcp__codex__codex-reply
to use mcp__llm-chat__chat instead, following the same pattern.
```


## 验证安装

### 1. 验证审查器端点

```bash
curl -s "https://openrouter.ai/api/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-or-v1-your-key" \
  -d '{
    "model": "openrouter/free",
    "messages": [{"role": "user", "content": "Say hello"}],
    "max_tokens": 50
  }'
```

预期：返回包含 `"choices"` 字段的 JSON。

### 2. 在 Claude Code 中端到端验证

```bash
claude
> 读一下这个项目，验证 /auto-review-loop-llm skill 是否正常可用
```

---

## 与其他方案对比

| | 默认方案 | Coding Plan | ModelScope | **OpenRouter** |
|---|---|---|---|---|
| 执行器 | Claude Opus | kimi-k2.5 | DeepSeek-V3 | 200+ 模型可选 |
| 审查器 | GPT-5.4 | glm-5 | DeepSeek-R1 | 200+ 模型可选 |
| 免费选项 | 无 | 无 | **有（2000次/天）** | **有（免费模型）** |
| API Key 数量 | 2 个 | 1 个 | 1 个 | **1 个** |
| 模型选择 | 受限 | 4 种 | 1000+ 种 | **200+ 种** |
| 价格 | 按量 | 套餐 | 免费 | 免费+按量 |

**OpenRouter 的独特优势**：一个 Key 访问所有主流模型，包括免费选项。

---

## 常见问题

**Q：免费模型有什么限制？**

免费模型有速率限制（通常 1-20 RPM），适合个人研究。大量使用时建议充值按量计费。

**Q：如何切换模型？**

修改 `settings.json` 中的 `LLM_MODEL` 值，重启 Claude Code 即可。

**Q：OpenRouter 支持 Anthropic 协议吗？**

支持！使用 `https://openrouter.ai/api/anthropic` 作为执行器端点。

**Q：为什么 llm-chat MCP 调用失败？**

检查：
1. API Key 格式正确（`sk-or-v1-` 开头）
2. 模型 ID 正确（含命名空间，如 `minimax/minimax-m2.5:free`）
3. 账户有足够额度（免费或充值）

---

## 参考资料

- [OpenRouter 官网](https://openrouter.ai/)
- [OpenRouter 模型列表](https://openrouter.ai/models)
- [OpenRouter 文档](https://openrouter.ai/docs)
- [LLM API 混搭配置指南](LLM_API_MIX_MATCH_GUIDE.md)
