# GitHub 提交通知接入飞书指南

## 效果预览

每次有人往仓库 push 代码或提 PR，飞书群会自动收到一条卡片消息：

**Push 通知（蓝色）**
```
📦 MOYAI1234/你的仓库名 有新提交
─────────────────────────────
分支          提交人
master        张三

提交信息
fix: 修复登录页面跳转异常

[查看提交]
```

**PR 通知（橙色/绿色/红色）**
```
🔀 新 PR 待审查
─────────────────────────────
仓库                  状态
MOYAI1234/你的仓库    新建

提交人          目标分支
张三            master

PR 标题
feat: 新增用户权限管理模块

[查看 PR]
```

---

## 接入步骤

> **前提**：你的仓库必须在 MOYAI1234 组织下。个人账号下的仓库无法使用组织的共享配置。

### 只需做一件事：在你的仓库里加一个文件

在仓库根目录创建以下文件（目录不存在就新建）：

**文件路径**：`.github/workflows/notify-feishu.yml`

**文件内容**（直接复制，不需要修改任何内容）：

```yaml
name: 飞书推送通知

on:
  push:
    branches: [master, main]
  pull_request:
    types: [opened, closed, reopened]

jobs:
  notify:
    uses: MOYAI1234/.github/.github/workflows/notify-feishu.yml@main
    with:
      event_name: ${{ github.event_name }}
      repo_name: ${{ github.repository }}
      actor: ${{ github.actor }}
      ref_name: ${{ github.ref_name }}
      commit_message: ${{ github.event.head_commit.message }}
      commit_url: ${{ github.event.head_commit.url }}
      pr_title: ${{ github.event.pull_request.title }}
      pr_url: ${{ github.event.pull_request.html_url }}
      pr_action: ${{ github.event.action }}
      pr_merged: ${{ toJson(github.event.pull_request.merged) }}
      pr_base: ${{ github.event.pull_request.base.ref }}
    secrets: inherit
```

文件提交并 push 后，下一次 push 或 PR 操作就会自动触发飞书通知，**无需其他任何配置**。

---

## 常见问题

**Q：我需要配置飞书的 Webhook 地址吗？**

不需要。飞书地址已经由管理员统一配置在组织级别，所有仓库自动继承，不用单独设置。

**Q：我想监听不同的分支怎么办？**

修改文件里的 `branches` 字段，比如改成 `[develop, release]`。

**Q：PR 合并也会通知吗？**

会。PR 被合并时会发一条绿色的"✅ PR 已合并"卡片；直接关闭（不合并）则是红色"❌ PR 已关闭"。

**Q：Actions 在哪里看运行记录？**

打开仓库页面 → 顶部点 **Actions** 标签 → 可以看到每次运行的日志和结果。

---

## 背后的原理（可选阅读）

### 为什么不直接用 GitHub 的 Webhook 功能？

GitHub 原生 Webhook 会把事件数据直接推给目标地址，但它发送的格式是 GitHub 自己定义的：

```json
{ "action": "push", "repository": {...}, "commits": [...] }
```

飞书机器人要求收到的消息必须包含 `msg_type` 字段：

```json
{ "msg_type": "interactive", "card": {...} }
```

两边格式不兼容，飞书收到后直接报错 `params error, msg_type need`。

**GitHub Actions 充当了中间的格式翻译层**：拿到 GitHub 的事件数据，重新拼成飞书能识别的格式，再通过 `curl` 发送出去。

### GitHub Actions 是怎么运行的？

Actions 是 GitHub 提供的自动化执行器，本质是事件驱动的：

```
你 push 代码
  → GitHub 检测到事件
  → 启动一台临时 Linux 服务器
  → 按照 .yml 文件逐步执行命令
  → 服务器销毁
```

整个过程在 GitHub 的服务器上完成，免费且不需要你自己维护服务器。

### 为什么你的文件只有这么短？

通知的完整逻辑（如何拼 JSON、如何区分 push 和 PR、卡片样式等）集中维护在组织的 `.github` 仓库中：

```
MOYAI1234 组织
├── .github 仓库          ← 完整的通知逻辑在这里（200 行）
│   └── .github/workflows/notify-feishu.yml
│
└── 你的仓库              ← 只有 15 行，调用上面的逻辑
    └── .github/workflows/notify-feishu.yml
```

你的文件里的 `uses: MOYAI1234/.github/...` 就是"去调用组织中央仓库里的工作流"。这样设计的好处是：**通知格式或飞书地址有变化，只需要改一个地方，所有仓库自动生效**。

### 飞书地址是怎么传进去的？

飞书的 Webhook 地址相当于一个密码，不能明文写在代码文件里（否则任何人都能往群里发消息）。

管理员已将其配置为组织级别的 Secret，存储在 GitHub 的加密环境中。工作流里的 `secrets: inherit` 表示"把当前仓库能访问的所有 Secret 都传给被调用的工作流"。

整体流程如下：

```
git push
  → Actions 启动
  → 你的 .yml 调用中央工作流，传入提交信息
  → 中央工作流从 Org Secret 读取飞书地址
  → curl 发送格式化的飞书卡片消息
  → 飞书群收到通知 ✅
```
