# Claude Code Action for YassLab team

YassLab チーム用の Claude Code カスタムアクションです。

[&raquo; デフォルト設定を見る](https://github.com/yasslab/claude_review_action/blob/main/action.yml)<br>
[&raquo; レビュー利用例を見る](https://github.com/coderdojo-japan/coderdojo.jp/pull/1719)<br>
[&raquo; 関連リポジトリを見る](https://github.com/yasslab/dependabot_auto_merge)

<br>

## カスタムアクションの設定例

```bash
.github/workflows/claude-review.yml
```

```yaml
name: Claude Review

on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
  pull_request:
    types: [opened, synchronize]
  pull_request_review:
    types: [submitted]
  pull_request_review_comment:
    types: [created]

jobs:
  claude-review:
    if: |
      (github.event_name == 'issues'                      && contains(github.event.issue.body,        '@claude')) ||
      (github.event_name == 'issue_comment'               && contains(github.event.comment.body,      '@claude')) ||
      (github.event_name == 'pull_request'                && contains(github.event.pull_request.body, '@claude')) ||
      (github.event_name == 'pull_request_review'         && contains(github.event.review.body,       '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body,      '@claude'))

    runs-on: ubuntu-latest

    # 推奨: 最小限の権限のみ付与
    permissions:
      contents:      read  # Repository 内の権限
      actions:       read  # Actionsログへの権限
      issues:        write # Issueコメントの権限 (id-token で昇格されるため明確化)
      pull-requests: write # PR 内コメントの権限 (id-token で昇格されるため明確化)
      id-token:      write # Claude Code Actions の実行に必要 (昇格する権限を持つ)

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Claude Review
        uses: yasslab/claude_review_action@main
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

<br>

## 実行条件

次の **両方** を満たす場合のみ Claude が動きます。いずれかを満たさない場合は、
コメント等を投稿せずに終了します（Actions のログには警告が残ります）。

1. イベントを起こしたユーザーが YassLab メンバーであること
2. 対象の Issue / PR の作成者が YassLab メンバー、または Dependabot であること

メンバー判定には [yasslab.jp/members.json](https://yasslab.jp/members.json) を使います。

2 を課しているのは、この Action が PR のブランチ（fork からの PR を含む）を runner に
チェックアウトし、レビュー中に `bundle install` などの Ruby コマンドを実行しうるためです。
作成者を確認しないと、メンバーがレビューを依頼した時点で、非メンバーが書いたコードが
runner 上で実行されてしまいます。実行を伴うツールを追加する際は、この前提が
崩れていないか確認してください。

Dependabot を例外にしているのは、fork ではなく対象リポジトリ内にブランチを作るため、
PR に含まれるのが write 権限保持者の管理下にある依存更新に限られるためです。

そのため、**外部コントリビュータからの PR はレビュー対象外**になります。

-----

Copyright &copy; [YassLab](http://github.com/yasslab).
