# 周报助手使用手册

自动从 GitHub 采集你的 PR 和 Issue 数据，生成一份给 leader 看的工作周报。

## 前置准备

### 1. 准备 GitHub Token

你需要一个 GitHub Personal Access Token（用于读取你的 PR 和 Issue 数据）。

1. 打开 https://github.com/settings/tokens/new
2. **Note** 随便填，如 `weekly-report`
3. **Expiration** 选 **No expiration**（或按需设置）
4. **勾选 `repo`** 权限（只需要这一个）
5. 点击 **Generate token**，复制生成的 token（以 `ghp_` 开头）

> Token 只会保存在你本地的 `~/.weekly-report/config.json` 中，不会上传到任何地方。

### 2. 安装 Skill

根据你使用的工具，选择对应的安装方式：

**Claude Code：**

```bash
cp -r weekly-report-skill ~/.claude/skills/weekly-report/
pip install -r ~/.claude/skills/weekly-report/requirements.txt
```

**Claw 容器（NanoClaw / OpenClaw / QClaw 等）：**

```bash
cp -r weekly-report-skill container/skills/weekly-report/
```

安装完成后，确认 Python 环境中有 `requests` 库：

```bash
pip install requests
```

## 使用方法

### 生成周报

直接说 **"周报"** 即可。

### 首次使用

首次使用时，助手会逐步引导你完成配置（只需一次）：

1. **GitHub Token** — 粘贴你准备好的 token
2. **GitHub 用户名** — 你的 GitHub 账号名
3. **岗位** — 如：后端研发、前端研发、产品经理、测试工程师、SRE 等
4. **仓库范围** — 默认采集 `matrixorigin` 组织，如果还有其他组织或个人仓库，告诉助手即可；如果只需要 matrixorigin，直接说"就这些"

配置会保存到本地 `~/.weekly-report/config.json`，之后每次说"周报"就会直接生成，不再重复询问。

### 日期范围

助手会自动判断日期范围：

| 今天是 | 生成的周报范围 |
|--------|--------------|
| 周四、周五 | 本周一 ~ 今天 |
| 周一、周二、周三 | 上周一 ~ 上周五 |

如果需要指定范围，直接说，比如"生成上周的周报"、"生成 3/10 到 3/14 的周报"。

### 采集范围

助手会采集以下数据：

- 你**创建**的 PR
- 你**评审**的 PR
- 你**参与**的 Issue（创建、评论、被 assign、被 mention）

不需要有关联 PR 的 Issue 也会被采集到。

### 周报风格

周报会根据你的岗位自动调整视角和侧重点，不使用固定模板。开头会有一段总结性概览，其中会突出风险、阻塞或延期的问题。

### 补充内容

生成周报后，助手会问"还有什么要补充的吗？"。你可以补充非 GitHub 上的工作，比如：

- "加上周三开了需求评审会"
- "补充：周二和客户做了方案对齐"
- "这周还做了 XX 的竞品调研"

助手会把补充内容融入周报重新输出。如果没有要补充的，说"没有了"即可。

## 修改配置

随时可以通过对话修改，比如：

- "我转岗了，现在是前端研发"
- "把 org:another-org 也加进来"
- "换个 token"
- "去掉 xxx 仓库"

## 输出方式

| 说法 | 效果 |
|------|------|
| "周报" | 直接在聊天中展示 |
| "生成文档" / "创建文档" | 保存为本地 markdown 文件 |

## 常见问题

**Q: Token 过期了怎么办？**
A: 助手会提示你 token 失效，重新生成一个发给它即可。

**Q: 周报里没有我的数据？**
A: 检查仓库范围是否包含了你工作的组织。可以说"看一下我的配置"确认。

**Q: 可以生成指定时间段的周报吗？**
A: 可以，直接说"生成 3/10 到 3/14 的周报"。

**Q: 我的数据安全吗？**
A: Token 和配置只保存在你本地的 `~/.weekly-report/config.json`，数据采集通过 GitHub API 直接获取，不经过任何第三方服务。
