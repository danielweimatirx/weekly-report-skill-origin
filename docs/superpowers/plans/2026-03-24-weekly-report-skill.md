# 周报助手 Phase 1 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个 Python CLI 工具采集 GitHub PR 数据并输出 JSON，配套 NanoClaw Container Skill 实现对话式周报生成。

**Architecture:** cli.py 负责确定性的数据采集（GitHub API 调用、分页、去重），输出结构化 JSON 到 stdout。SKILL.md 定义 NanoClaw agent 的行为指令，负责日期推断、调用 CLI、AI 总结和输出格式化。两者通过 JSON 管道解耦。

**Tech Stack:** Python 3.10+, requests, pytest, NanoClaw Container Skill (SKILL.md)

**Spec:** `docs/superpowers/specs/2026-03-24-daily-report-skill-design.md`

---

## 文件结构

```
daily-report/
├── cli.py                  # GitHub PR 数据采集 CLI（~150 行）
├── tests/
│   └── test_cli.py         # CLI 单元测试
├── requirements.txt        # requests
├── SKILL.md                # NanoClaw Container Skill
└── docs/superpowers/...    # 设计文档和计划（已有）
```

---

### Task 1: CLI 骨架 — 参数解析与 main 入口

**Files:**
- Create: `cli.py`
- Create: `tests/test_cli.py`

- [ ] **Step 1: 写失败测试 — 参数解析**

```python
# tests/test_cli.py
import subprocess
import json

def test_cli_missing_required_args():
    """缺少必需参数时应退出码 2"""
    result = subprocess.run(
        ["python", "cli.py"],
        capture_output=True, text=True
    )
    assert result.returncode == 2

def test_cli_help():
    """--help 应正常输出"""
    result = subprocess.run(
        ["python", "cli.py", "--help"],
        capture_output=True, text=True
    )
    assert result.returncode == 0
    assert "--user" in result.stdout
    assert "--org" in result.stdout
    assert "--since" in result.stdout
    assert "--until" in result.stdout
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `cd /Users/zhangqq/Documents/pythonProject/dailyReport && python -m pytest tests/test_cli.py -v`
Expected: FAIL — `cli.py` 不存在

- [ ] **Step 3: 实现 CLI 骨架**

```python
# cli.py
"""GitHub PR 数据采集 CLI，输出结构化 JSON。"""

import argparse
import json
import sys


def parse_args(argv=None):
    parser = argparse.ArgumentParser(description="采集 GitHub PR 数据，输出 JSON")
    parser.add_argument("--user", required=True, help="GitHub 用户名")
    parser.add_argument("--org", required=True, help="GitHub 组织名")
    parser.add_argument("--since", required=True, help="开始日期（含），格式 YYYY-MM-DD")
    parser.add_argument("--until", required=True, help="结束日期（含），格式 YYYY-MM-DD")
    parser.add_argument("--token", default=None, help="GitHub token，默认读取 GITHUB_TOKEN 环境变量")
    return parser.parse_args(argv)


def main(argv=None):
    args = parse_args(argv)
    # TODO: 实现数据采集
    json.dump([], sys.stdout, ensure_ascii=False, indent=2)
    print()


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `cd /Users/zhangqq/Documents/pythonProject/dailyReport && python -m pytest tests/test_cli.py -v`
Expected: 2 PASS

- [ ] **Step 5: 提交**

```bash
git add cli.py tests/test_cli.py
git commit -m "feat: cli skeleton with argument parsing"
```

---

### Task 2: GitHub API 请求封装

**Files:**
- Modify: `cli.py`
- Modify: `tests/test_cli.py`

- [ ] **Step 1: 写失败测试 — API 请求函数**

```python
# tests/test_cli.py（追加）
import pytest
from unittest.mock import patch, MagicMock
import cli

def _mock_response(status_code, json_data, headers=None):
    resp = MagicMock()
    resp.status_code = status_code
    resp.json.return_value = json_data
    resp.text = json.dumps(json_data)
    resp.headers = headers or {"X-RateLimit-Remaining": "29", "X-RateLimit-Reset": "0"}
    return resp

@patch("cli.requests.get")
def test_search_prs_basic(mock_get):
    """正常返回 PR 列表"""
    mock_get.return_value = _mock_response(200, {
        "total_count": 1,
        "items": [{
            "number": 100,
            "title": "feat: add feature",
            "state": "open",
            "created_at": "2026-03-17T10:00:00Z",
            "pull_request": {"merged_at": None},
            "html_url": "https://github.com/org/repo/pull/100",
            "repository_url": "https://api.github.com/repos/org/repo",
        }]
    })
    results = cli.search_prs("testuser", "org", "2026-03-17", "2026-03-21", "fake-token")
    assert len(results) == 1
    assert results[0]["pr_number"] == 100
    assert results[0]["repo"] == "repo"

@patch("cli.requests.get")
def test_search_prs_auth_error(mock_get):
    """401 应返回错误字典"""
    mock_get.return_value = _mock_response(401, {"message": "Bad credentials"})
    results = cli.search_prs("testuser", "org", "2026-03-17", "2026-03-21", "bad-token")
    assert "error" in results
    assert results["error"] == "auth_failed"

@patch("cli.requests.get")
def test_search_prs_no_token(mock_get):
    """未提供 token 时应返回 auth_failed"""
    results = cli.search_prs("testuser", "org", "2026-03-17", "2026-03-21", token=None)
    assert results["error"] == "auth_failed"

@patch("cli.requests.get")
def test_search_prs_empty_result(mock_get):
    """无结果时返回空列表"""
    mock_get.return_value = _mock_response(200, {"total_count": 0, "items": []})
    results = cli.search_prs("testuser", "org", "2026-03-17", "2026-03-21", "fake-token")
    assert results == []

@patch("cli.requests.get")
def test_search_prs_pagination(mock_get):
    """分页：第一页 100 条，第二页剩余"""
    page1_items = [{"number": i, "title": f"pr-{i}", "state": "open",
                    "created_at": "2026-03-17T10:00:00Z",
                    "pull_request": {"merged_at": None},
                    "html_url": f"https://github.com/org/repo/pull/{i}",
                    "repository_url": "https://api.github.com/repos/org/repo"}
                   for i in range(100)]
    page2_items = [{"number": 100, "title": "pr-100", "state": "open",
                    "created_at": "2026-03-17T10:00:00Z",
                    "pull_request": {"merged_at": None},
                    "html_url": "https://github.com/org/repo/pull/100",
                    "repository_url": "https://api.github.com/repos/org/repo"}]
    mock_get.side_effect = [
        _mock_response(200, {"total_count": 101, "items": page1_items}),
        _mock_response(200, {"total_count": 101, "items": page2_items}),
    ]
    results = cli.search_prs("testuser", "org", "2026-03-17", "2026-03-21", "fake-token")
    assert len(results) == 101

@patch("cli.time.sleep")
@patch("cli.requests.get")
def test_search_prs_rate_limit_retry(mock_get, mock_sleep):
    """429 限流后重试成功"""
    mock_get.side_effect = [
        _mock_response(429, {"message": "rate limit"}, {"X-RateLimit-Remaining": "0", "X-RateLimit-Reset": "0"}),
        _mock_response(200, {"total_count": 1, "items": [{
            "number": 1, "title": "pr", "state": "open",
            "created_at": "2026-03-17T10:00:00Z",
            "pull_request": {"merged_at": None},
            "html_url": "https://github.com/org/repo/pull/1",
            "repository_url": "https://api.github.com/repos/org/repo"}]}),
    ]
    results = cli.search_prs("testuser", "org", "2026-03-17", "2026-03-21", "fake-token")
    assert len(results) == 1
    mock_sleep.assert_called_once()
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `python -m pytest tests/test_cli.py::test_search_prs_basic tests/test_cli.py::test_search_prs_auth_error -v`
Expected: FAIL — `search_prs` 不存在

- [ ] **Step 3: 实现 search_prs**

在 `cli.py` 中添加：

在 `cli.py` 顶部 import 区添加（与 argparse、json、sys 并列）：

```python
import os
import time
from datetime import datetime, timedelta
import requests
```

然后添加函数：

```python
GITHUB_API = "https://api.github.com"
RATE_LIMIT_BUFFER = 5  # 剩余次数低于此值时主动等待


def search_prs(user, org, since, until, token, query_prefix="author"):
    """搜索 GitHub PR，返回列表或错误字典。

    query_prefix: "author" 或 "reviewed-by"
    token: 必需，调用者负责传入
    """
    if not token:
        return {"error": "auth_failed", "message": "未提供 GitHub token"}

    # until +1 天，转为左闭右开区间
    until_exclusive = (datetime.strptime(until, "%Y-%m-%d") + timedelta(days=1)).strftime("%Y-%m-%d")

    query = f"{query_prefix}:{user} org:{org} type:pr updated:{since}T00:00:00+08:00..{until_exclusive}T00:00:00+08:00"
    headers = {"Authorization": f"token {token}", "Accept": "application/vnd.github.v3+json"}

    all_items = []
    page = 1

    while True:
        for attempt in range(3):
            resp = requests.get(
                f"{GITHUB_API}/search/issues",
                params={"q": query, "per_page": 100, "page": page, "sort": "updated", "order": "desc"},
                headers=headers,
            )

            # 缓存 JSON 解析结果，避免重复调用 + 防止非 JSON 响应崩溃
            try:
                body = resp.json()
            except Exception:
                body = {}
            msg = body.get("message", "")

            if resp.status_code == 200:
                # 主动检查 rate limit 余量
                remaining = int(resp.headers.get("X-RateLimit-Remaining", "99"))
                if remaining < RATE_LIMIT_BUFFER:
                    reset_at = int(resp.headers.get("X-RateLimit-Reset", "0"))
                    wait = max(1, reset_at - int(time.time()))
                    time.sleep(min(wait, 30))
                break
            elif resp.status_code in (401, 403) and "rate limit" not in msg.lower():
                return {"error": "auth_failed", "message": msg or "认证失败"}
            elif resp.status_code == 429 or (resp.status_code == 403 and "rate limit" in msg.lower()):
                reset_at = int(resp.headers.get("X-RateLimit-Reset", "0"))
                wait = max(1, reset_at - int(time.time()))
                time.sleep(min(wait, 30))
            else:
                return {"error": "github_unreachable", "message": f"GitHub API 返回 {resp.status_code}: {resp.text[:200]}"}
        else:
            return {"error": "github_unreachable", "message": "GitHub API 请求失败，已重试 3 次"}

        items = body.get("items", [])
        for item in items:
            repo_name = item["repository_url"].rsplit("/", 1)[-1]
            all_items.append({
                "repo": repo_name,
                "pr_number": item["number"],
                "title": item["title"],
                "state": "merged" if item.get("pull_request", {}).get("merged_at") else item["state"],
                "role": [query_prefix.replace("-", "_")],  # "author" 或 "reviewed_by"
                "created_at": item["created_at"][:10],
                "merged_at": (item.get("pull_request", {}).get("merged_at") or "")[:10] or None,
                "url": item["html_url"],
            })

        if len(items) < 100:
            break
        page += 1

    return all_items
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `python -m pytest tests/test_cli.py -v`
Expected: 全部 PASS

- [ ] **Step 5: 提交**

```bash
git add cli.py tests/test_cli.py
git commit -m "feat: GitHub search API request with pagination and error handling"
```

---

### Task 3: PR 去重与角色合并

**Files:**
- Modify: `cli.py`
- Modify: `tests/test_cli.py`

- [ ] **Step 1: 写失败测试 — 去重逻辑**

```python
# tests/test_cli.py（追加）
def test_merge_and_dedupe():
    """同一 PR 既是 author 又是 reviewer 时，合并角色"""
    authored = [
        {"repo": "matrixflow", "pr_number": 100, "title": "feat: x", "state": "open",
         "role": ["author"], "created_at": "2026-03-17", "merged_at": None,
         "url": "https://github.com/org/matrixflow/pull/100"},
    ]
    reviewed = [
        {"repo": "matrixflow", "pr_number": 100, "title": "feat: x", "state": "open",
         "role": ["reviewed_by"], "created_at": "2026-03-17", "merged_at": None,
         "url": "https://github.com/org/matrixflow/pull/100"},
        {"repo": "matrixflow", "pr_number": 200, "title": "fix: y", "state": "merged",
         "role": ["reviewed_by"], "created_at": "2026-03-18", "merged_at": "2026-03-19",
         "url": "https://github.com/org/matrixflow/pull/200"},
    ]
    result = cli.merge_and_dedupe(authored, reviewed)
    assert len(result) == 2
    pr100 = next(r for r in result if r["pr_number"] == 100)
    assert sorted(pr100["role"]) == ["author", "reviewed_by"]
    pr200 = next(r for r in result if r["pr_number"] == 200)
    assert pr200["role"] == ["reviewed_by"]
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `python -m pytest tests/test_cli.py::test_merge_and_dedupe -v`
Expected: FAIL — `merge_and_dedupe` 不存在

- [ ] **Step 3: 实现去重函数**

在 `cli.py` 中添加：

```python
def merge_and_dedupe(authored, reviewed):
    """合并 authored 和 reviewed 的 PR 列表，按 repo+pr_number 去重，合并角色。"""
    index = {}
    for pr in authored + reviewed:
        key = (pr["repo"], pr["pr_number"])
        if key in index:
            existing_roles = set(index[key]["role"])
            existing_roles.update(pr["role"])
            index[key]["role"] = sorted(existing_roles)
        else:
            index[key] = dict(pr)
    return sorted(index.values(), key=lambda x: x["created_at"], reverse=True)
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `python -m pytest tests/test_cli.py -v`
Expected: 全部 PASS

- [ ] **Step 5: 提交**

```bash
git add cli.py tests/test_cli.py
git commit -m "feat: PR deduplication with role merging"
```

---

### Task 4: 串联 main — 完整 CLI 流程

**Files:**
- Modify: `cli.py`
- Modify: `tests/test_cli.py`

- [ ] **Step 1: 写失败测试 — 端到端 main 调用**

```python
# tests/test_cli.py（追加）
@patch("cli.requests.get")
def test_main_end_to_end(mock_get, capsys):
    """完整 main 调用，输出 JSON"""
    mock_get.return_value = _mock_response(200, {
        "total_count": 1,
        "items": [{
            "number": 42,
            "title": "fix(catalog): proxy mode issue",
            "state": "closed",
            "created_at": "2026-03-17T08:00:00Z",
            "pull_request": {"merged_at": "2026-03-18T10:00:00Z"},
            "html_url": "https://github.com/matrixorigin/matrixflow/pull/42",
            "repository_url": "https://api.github.com/repos/matrixorigin/matrixflow",
        }]
    })
    cli.main(["--user", "aqqi666", "--org", "matrixorigin",
              "--since", "2026-03-17", "--until", "2026-03-21", "--token", "fake"])
    captured = capsys.readouterr()
    data = json.loads(captured.out)
    assert len(data) >= 1
    assert data[0]["pr_number"] == 42
    assert data[0]["state"] == "merged"

@patch("cli.requests.get")
def test_main_error_output(mock_get, capsys):
    """API 错误时输出错误 JSON，退出码 1"""
    mock_get.return_value = _mock_response(401, {"message": "Bad credentials"})
    with pytest.raises(SystemExit) as exc_info:
        cli.main(["--user", "x", "--org", "y",
                  "--since", "2026-03-17", "--until", "2026-03-21", "--token", "bad"])
    assert exc_info.value.code == 1
    captured = capsys.readouterr()
    data = json.loads(captured.out)
    assert "error" in data

@patch("cli.requests.get")
def test_main_empty_result(mock_get, capsys):
    """无 PR 时输出空数组，退出码 0"""
    mock_get.return_value = _mock_response(200, {"total_count": 0, "items": []})
    cli.main(["--user", "x", "--org", "y",
              "--since", "2026-03-17", "--until", "2026-03-21", "--token", "fake"])
    captured = capsys.readouterr()
    data = json.loads(captured.out)
    assert data == []
```

- [ ] **Step 2: 运行测试，确认失败**

Run: `python -m pytest tests/test_cli.py::test_main_end_to_end tests/test_cli.py::test_main_error_output tests/test_cli.py::test_main_empty_result -v`
Expected: FAIL — main 还是输出空数组

- [ ] **Step 3: 串联 main 函数**

更新 `cli.py` 的 `main` 函数：

```python
def main(argv=None):
    args = parse_args(argv)
    token = args.token or os.environ.get("GITHUB_TOKEN")

    # 采集 authored PR
    authored = search_prs(args.user, args.org, args.since, args.until, token, query_prefix="author")
    if isinstance(authored, dict) and "error" in authored:
        json.dump(authored, sys.stdout, ensure_ascii=False, indent=2)
        print()
        sys.exit(1)

    # 采集 reviewed PR
    reviewed = search_prs(args.user, args.org, args.since, args.until, token, query_prefix="reviewed-by")
    if isinstance(reviewed, dict) and "error" in reviewed:
        json.dump(reviewed, sys.stdout, ensure_ascii=False, indent=2)
        print()
        sys.exit(1)

    # 合并去重
    result = merge_and_dedupe(authored, reviewed)

    json.dump(result, sys.stdout, ensure_ascii=False, indent=2)
    print()
```

- [ ] **Step 4: 运行测试，确认通过**

Run: `python -m pytest tests/test_cli.py -v`
Expected: 全部 PASS

注意：测试直接调用 `cli.main()` 而非 subprocess，这样 mock 才能生效。

- [ ] **Step 5: 提交**

```bash
git add cli.py tests/test_cli.py
git commit -m "feat: wire up main with full fetch-merge-output pipeline"
```

---

### Task 5: requirements.txt 与本地真实测试

**Files:**
- Create: `requirements.txt`

- [ ] **Step 1: 创建依赖文件**

`requirements.txt`（运行时依赖）：
```
requests
```

`requirements-dev.txt`（开发依赖）：
```
-r requirements.txt
pytest
```

- [ ] **Step 2: 安装依赖**

Run: `cd /Users/zhangqq/Documents/pythonProject/dailyReport && pip install -r requirements-dev.txt`

- [ ] **Step 3: 运行全部单元测试**

Run: `python -m pytest tests/ -v`
Expected: 全部 PASS

- [ ] **Step 4: 真实 API 调用测试（手动验证）**

Run: `python cli.py --user aqqi666 --org matrixorigin --since 2026-03-17 --until 2026-03-24`
Expected: 输出包含近期 PR 的 JSON 数组（使用本机 GITHUB_TOKEN 环境变量）

- [ ] **Step 5: 提交**

```bash
git add requirements.txt requirements-dev.txt
git commit -m "chore: add requirements and dev dependencies"
```

---

### Task 6: NanoClaw Container Skill

**Files:**
- Create: `SKILL.md`

- [ ] **Step 1: 编写 SKILL.md**

```markdown
---
name: weekly-report
description: 生成工作周报。采集 GitHub PR 数据，用中文总结每个 PR，支持对话补充内容。说"周报"即可触发。
---

# 周报助手

你是一个周报生成助手。用户说"周报"或类似意图时，按以下流程工作。

## 1. 获取用户 GitHub 信息

检查 CLAUDE.md 记忆中是否有用户的 GitHub 用户名。

- 如果没有，询问："你的 GitHub 用户名是？"，用户回答后记到 CLAUDE.md 中。
- 如果有，直接使用。

默认组织为 `matrixorigin`。

## 2. 推断日期范围

根据今天是星期几推断默认周期：

- **周四、周五**：本周周报，范围 = 本周一 ~ 今天
- **周一、周二、周三**：补上周周报，范围 = 上周一 ~ 上周五

如果用户指定了范围（如"上周的"、"这周的"），以用户为准。

## 3. 采集数据

运行以下命令采集 PR 数据：

```bash
python ${CLAUDE_SKILL_DIR}/cli.py --user {github_username} --org matrixorigin --since {since} --until {until} --token {token}
```

其中 `{token}` 从 host 侧配置读取（由 NanoClaw 框架注入），不要从容器环境变量中读取。

解析 stdout 的 JSON 输出。

如果返回的 JSON 包含 `error` 字段，向用户说明原因（如"GitHub API 暂时不可用，请稍后再试"），**不要编造数据**。

## 4. 生成周报

将每个 PR 用中文重新描述，要求：

- 通俗易懂，不是简单翻译 PR 标题
- 技术术语保留英文（如 S3、gRPC、MCP、MOWL 等）
- 按 PR 逐条列出
- 每条包含：仓库名、PR 编号（带链接）、状态（进行中/已合并/已关闭）、你的角色（作者/Reviewer）
- 如果用户同时是作者和 Reviewer，标注"作者 & Reviewer"

输出格式示例：

```
## 周报 2026.03.17 - 2026.03.21

1. **matrixflow #8740** - 为 MOWL 工作流执行添加了端到端测试（进行中，作者）
2. **matrixflow #8724** - 将 insight 模块的语义后处理规则从硬编码改为从知识库查询（进行中，作者）
3. **matrixflow #8715** - 修复 catalog 代理模式下缺失的 gRPC 方法和 S3 兼容性问题（已合并，作者）
4. **gitops #356** - 更新部署配置的资源限制参数（已合并，Reviewer）
```

## 5. 补充内容

检查 CLAUDE.md 记忆中的用户偏好：

- 如果偏好为"每次询问"：生成周报后问"还有什么要补充的吗？"
- 否则（默认）：直接输出周报，不主动询问

用户随时可以主动补充，如"加上周三开了需求评审会"，你应将其加入周报并重新输出。

如果用户说"以后每次都问我要不要补充"，记到 CLAUDE.md 中。
如果用户说"不用问了"，从 CLAUDE.md 中移除该偏好。

## 6. 输出

默认在聊天中直接展示周报。

如果用户要求"生成文档"或"创建文档"：
- 有企业微信文档 MCP 能力时：调用文档接口创建企业微信文档
- 没有时：保存为本地 markdown 文件，告知用户文件路径

如果用户要求"生成表格"或"创建表格"：
- 有企业微信智能表格 MCP 能力时：创建智能表格，每行一个 PR，列为：仓库、PR 编号、描述、状态、角色
- 没有时：提示用户需要接入企业微信后才可使用此功能
```

- [ ] **Step 2: 验证 SKILL.md 格式**

检查：
- frontmatter 包含 `name` 和 `description`
- name 是 lowercase + hyphens
- 内容 < 500 行

- [ ] **Step 3: 提交**

```bash
git add SKILL.md
git commit -m "feat: add NanoClaw weekly-report container skill"
```

---

### Task 7: 端到端验证

- [ ] **Step 1: 用真实数据运行 CLI**

Run: `python cli.py --user aqqi666 --org matrixorigin --since 2026-03-17 --until 2026-03-24`

验证：
- 输出为合法 JSON 数组
- 包含已知的 PR（如 #8740、#8724、#8715）
- role、state、repo 字段正确

- [ ] **Step 2: 全部测试通过**

Run: `python -m pytest tests/ -v`
Expected: 全部 PASS

- [ ] **Step 3: 最终提交**

确保工作区干净：`git status`
