### 代码目录
```
pusher/
├── .github/
│   └── workflows/
│       ├── notify-telegram.yml   # TG推送模板
└── README.md                     # 使用说明
```

### 配置仓库全局 Secrets（仅需 1 次）   
- 进入仓库 → Settings → Secrets and variables → Actions → 添加所有推送方式的密钥（后续所有仓库复用）

| Secret名称 | 说明 | 示例值 |
| ---- | ---- | ---- |
| TG_BOT_TOKEN | TG机器人Token | 123456:ABC-DEF1234ghIkl-zyx57W2v1u |

### 编写各推送方式的可复用模板
- 所有模板均标记为 workflow_call，支持外部仓库传入「标题、内容」参数，单独调用任意推送能力

### 其他仓库单独调用任意推送模板
- 在任意仓库的工作流中，可单独调用某一个推送模板，也可组合调用多个模板，无需重复配置密钥。
- 在目标仓库创建 .github/workflows/test-tg-notify.yml：
```
name: Test TG Notify

# 手动触发测试
on:
  workflow_dispatch:

jobs:
  # 调用TG推送模板
  tg-notify:
    # 格式：用户名/模板仓库名/.github/workflows/模板文件名@分支
    uses: 你的GitHub用户名/github-notify-templates/.github/workflows/notify-telegram.yml@main
    with:
      # 传入推送标题和内容（自定义）
      notify_title: "测试TG推送"
      notify_content: "这是从其他仓库调用中心化模板的测试消息\n- 时间：$(date +'%Y-%m-%d %H:%M:%S')\n- 仓库：${{ github.repository }}"
    secrets:
      # 传递PAT_TOKEN（跨仓库调用权限，模板仓库已配置）
      PAT_TOKEN: ${{ secrets.PAT_TOKEN }}
```
- 组合调用 TG + 钉钉推送模板
```
name: Test Multi Notify (TG + DingTalk)

on:
  workflow_dispatch:

jobs:
  # 调用TG推送
  tg-notify:
    uses: 你的用户名/github-notify-templates/.github/workflows/notify-telegram.yml@main
    with:
      notify_title: "任务执行结果"
      notify_content: "✅ 数据同步完成\n📝 执行日志：${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
    secrets:
      PAT_TOKEN: ${{ secrets.PAT_TOKEN }}

  # 调用钉钉推送（与TG并行执行）
  dingtalk-notify:
    uses: 你的用户名/github-notify-templates/.github/workflows/notify-dingtalk.yml@main
    with:
      notify_title: "任务执行结果"
      notify_content: "✅ 数据同步完成\n📝 执行日志：${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
    secrets:
      PAT_TOKEN: ${{ secrets.PAT_TOKEN }}
```
- 结合 Fork 同步逻辑调用推送
```
name: Fork Sync + Multi Notify

on:
  schedule:
    - cron: "0 0 * * *"
  workflow_dispatch:

jobs:
  # 第一步：执行Fork同步
  sync-fork:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    outputs:
      sync_status: ${{ steps.sync.outcome }}  # 输出同步状态供后续使用
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Sync Upstream
        id: sync
        uses: aormsby/Fork-Sync-With-Upstream-action@v3.4
        with:
          upstream_sync_repo: "原作者/仓库名"
          upstream_sync_branch: "main"
          target_sync_branch: "main"
          target_repo_token: ${{ secrets.PAT_TOKEN }}

  # 第二步：调用TG推送同步结果（依赖同步步骤完成）
  tg-notify:
    needs: sync-fork  # 依赖同步步骤
    uses: 你的用户名/github-notify-templates/.github/workflows/notify-telegram.yml@main
    with:
      notify_title: ${{ needs.sync-fork.outputs.sync_status == 'success' && '✅ Fork同步成功' || '❌ Fork同步失败' }}
      notify_content: "仓库：${{ github.repository }}\n分支：main\n状态：${{ needs.sync-fork.outputs.sync_status }}\n日志：${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
    secrets:
      PAT_TOKEN: ${{ secrets.PAT_TOKEN }}
```
### 核心优势
- 完全解耦：推送能力与业务逻辑（如 Fork 同步、CI/CD）分离，可单独调用任意推送方式；
- 全局复用：所有推送密钥仅在模板仓库配置 1 次，其他仓库无需重复设置；
- 低维护成本：修改推送逻辑（如 TG 消息格式）时，只需改模板仓库，所有引用仓库自动生效；
- 灵活组合：支持单独调用、组合调用，适配不同场景的推送需求；
- 最小权限：模板仅申请必要权限，符合 GitHub 安全规范。

### 注意事项
- PAT_TOKEN 权限：需勾选 repo 和 workflow 权限，确保跨仓库调用模板；
- 密钥优先级：若目标仓库配置了同名 Secrets，会覆盖模板仓库的全局 Secrets（可用于个性化配置）；
- 网络适配：部分推送方式（如企业微信）需确保 GitHub Actions 能访问对应 API（无需代理，默认可访问）；
- 失败处理：模板内置推送结果检查，失败时会输出明确提示，便于排查。
