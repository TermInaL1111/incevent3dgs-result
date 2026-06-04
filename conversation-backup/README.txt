# Claude Code 对话备份

## 恢复方法

1. 新服务器安装 Claude Code:
   curl -fsSL https://claude.ai/code/install.sh | bash

2. 恢复会话:
   把 claude-sessions/ 目录复制到新服务器的 ~/.claude/projects/-root/
   把 settings.json 复制到 ~/.claude/settings.json

3. 打开 Claude Code 后, 对话历史会自动加载

## 文件说明
- claude-sessions/*.jsonl — 完整对话记录
- settings.json — Claude Code 配置

