# claude code 最佳实践

具有基础ai coding使用经验的使用者可以快速上手cc，此处使用claude code cli + deepseek v4系列

## 安装claude code

```shell
npm install -g @anthropic-ai/claude-code
```

## 配置 deepseek-v4（windows）

```shell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="<你的 DeepSeek API Key>"
$env:ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_SUBAGENT_MODEL="deepseek-v4-flash"
$env:CLAUDE_CODE_EFFORT_LEVEL="max"
```

## claude code最佳实践

[官网最佳实践](https://code.claude.com/docs/zh-CN/best-practices)



