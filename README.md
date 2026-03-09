# MOYAI1234 组织共享配置

本仓库是组织级别的共享配置中心，存放所有仓库通用的 GitHub Actions 工作流和团队文档。

---

## 目录结构

```
.github/
├── .github/workflows/
│   └── notify-feishu.yml     # 飞书通知可复用工作流（所有仓库共享）
├── FEISHU_NOTIFY_GUIDE.md    # 飞书通知接入指南
└── README.md                 # 本文件
```

---

## 飞书推送通知

所有仓库的飞书通知逻辑统一维护在 `.github/workflows/notify-feishu.yml`。

触发事件：
- **Push**：推送代码到 master / main 分支时通知
- **PR**：PR 新建、合并、关闭时通知

新仓库接入只需在仓库内添加一个 `.github/workflows/notify-feishu.yml`，内容如下：

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

详细接入说明见 [FEISHU_NOTIFY_GUIDE.md](./FEISHU_NOTIFY_GUIDE.md)。

---

## 注意事项

- 本仓库必须保持 **Public**，否则其他仓库无法调用可复用工作流
- 修改 `notify-feishu.yml` 后所有仓库自动生效，无需逐个更新
- 私有仓库需在各自的 Repository Settings 中单独添加 `FEISHU_WEBHOOK_URL` Secret
