# weekly-report-skill

周报生成 Skill，支持各类 Claw 容器（NanoClaw、OpenClaw、QClaw 等）及 Claude Code。从 GitHub 采集 PR 和 Issue 数据，自动生成给 leader 看的工作周报。

## 功能

- 采集指定用户在组织下的 PR（authored + reviewed）和 Issue（involved）
- PR 自动关联 Issue（通过 body 中的 `fixes #xxx` / `closes #xxx`）
- 已合并 PR 附带代码变更统计（additions / deletions / changed_files）
- 按业务主题自动分类，生成结构化周报
- 支持对话式补充非 GitHub 内容（会议、评审等）

## 使用方式

### 作为 Claw Container Skill

将本目录复制到 Claw 容器的 `container/skills/weekly-report/`（适用于 NanoClaw、OpenClaw、QClaw 等），然后对机器人说"周报"。

### 作为 Claude Code Skill

```bash
# 复制到 Claude Code skills 目录
cp -r . ~/.claude/skills/weekly-report/

# 安装依赖
pip install -r requirements.txt
```

然后在 Claude Code 里说"周报"。

### 独立使用 CLI

```bash
pip install -r requirements.txt

# 使用 gh CLI 的 token
GITHUB_TOKEN=$(gh auth token) python3 cli.py \
  --user your-github-username \
  --org your-org \
  --since 2026-03-16 \
  --until 2026-03-20
```

输出结构化 JSON（`{"prs": [...], "issues": [...]}`），可配合 AI 工具生成周报。

## 配置

| 参数 | 说明 |
|------|------|
| `--user` | GitHub 用户名 |
| `--org` | GitHub 组织名 |
| `--since` | 开始日期（含），YYYY-MM-DD |
| `--until` | 结束日期（含），YYYY-MM-DD |
| `--token` | GitHub token，也可通过 `GITHUB_TOKEN` 环境变量设置 |

## 文件说明

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Skill 定义，描述 agent 的行为指令 |
| `cli.py` | GitHub 数据采集 CLI，输出 JSON |
| `requirements.txt` | Python 依赖（requests） |
