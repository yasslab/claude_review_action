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

1. **イベントを起こした人**が名簿に載っていて、GitHub の数値 ID が一致すること
2. **対象の Issue / PR を作成した人**が名簿に載っていて ID が一致すること
   （Dependabot だけは例外。後述）

名簿は [`action.yml`](action.yml) の `MEMBER_IDS` です。**ここが唯一の認可の根拠**で、
外部サイトへの問い合わせはありません。

名前ではなく数値 ID で照合しているのは、GitHub のユーザー名が改名なら即座に、退会なら
90 日後に解放され、第三者が取得できるためです。名前だけで判定していると、解放された
名前を取得した第三者が通過してしまいます。数値 ID は不変で再利用されないため、同じ
名前でも ID が違えば別人と判定できます。

2 を課しているのは、この Action が PR のブランチ（fork からの PR を含む）を runner に
チェックアウトし、レビュー中に `bundle install` などの Ruby コマンドを実行しうるためです。
作成者を確認しないと、メンバーがレビューを依頼した時点で、非メンバーが書いたコードが
runner 上で実行されてしまいます。実行を伴うツールを追加する際は、この前提が
崩れていないか確認してください。

そのため、**外部コントリビュータからの PR はレビュー対象外**になります。

### Dependabot の扱い

**メンバーが Dependabot の PR にレビューを依頼する**運用があるため、作成者としてのみ
許可しています（ID `49699333` も照合）。一方、**Dependabot 自身が Claude を起動する**
使い方は想定していないため、イベントを起こした人としては拒否されます。

### メンバーを追加するとき

`action.yml` の `MEMBER_IDS` に 1 行追加してください。

```bash
gh api users/<username> --jq .id
```

大文字小文字は GitHub の正式表記に合わせてください（完全一致で照合します）。
**未登録のユーザーは実行できません。** これは意図的な設計で、追加漏れを「動かない」
という形で必ず発覚させるためです。エラーメッセージに追加方法が出ます。

名簿から外すときも、同じく `MEMBER_IDS` の該当行を編集してください。

なお、Claude が読み込むコメントは**メンバーと Dependabot が書いたものだけ**に絞っています。
非メンバーのコメントが混ざると、そこに書かれた文章を指示として解釈してしまう余地が
残るためです（プロンプトインジェクション）。

<br>

## 使い方

`@claude` に**具体的な指示を書けば、その指示に従います**。

```
@claude この PR の N+1 問題を洗い出して、修正案を提示してください
```

指示を書かずにメンションした場合は、既定のコードレビューを行います。

```
@claude
@claude review this
```

いずれの場合も、日本語で回答し、レビューは `<details>` の折りたたみで整理されます。

**コミット・push はできません。** ローカルでファイルを編集して動作を確かめることは
できますが、変更はコメント内に差分として提示されます。実装を取り込む場合は、
その差分を手元で適用してください。

-----

Copyright &copy; [YassLab](http://github.com/yasslab).
