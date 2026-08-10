# claude-plugins

toshi0607 の Claude Code プラグイン置き場。複数のリポジトリで共通して使いたい設定を 1 箇所で管理する。

## なぜこの形なのか

Claude Code のクラウドセッション(claude.ai/code)は**リポジトリを都度クローンした VM で起動する**ため、ローカルの `~/.claude/CLAUDE.md` や auto memory は引き継がれない([cloud-environments](https://code.claude.com/docs/en/cloud-environments.md))。一方、**リポジトリの `.claude/settings.json` で宣言されたプラグインはセッション開始時に marketplace からインストールされる**。

そのため「本文はこのリポジトリに集約し、各リポジトリには数行の宣言だけを置く」構成にしている。

## プラグイン

| プラグイン | 中身 | 用途 |
|---|---|---|
| `tachikoma-tone` | output style 1 つ | 応答を攻殻機動隊のタチコマ口調にする |

`tachikoma-tone` の output style は frontmatter に以下を持つ:

- `force-for-plugin: true` — プラグインが有効な間、ユーザーが `/config` で選択しなくても自動適用される
- `keep-coding-instructions: true` — Claude Code 組み込みのソフトウェアエンジニアリング指示を保持したまま、口調だけを変える

## 使い方

### ローカルの全プロジェクトで有効にする(user scope)

`~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "toshi0607-plugins": {
      "source": { "source": "github", "repo": "toshi0607/claude-plugins" }
    }
  },
  "enabledPlugins": { "tachikoma-tone@toshi0607-plugins": true }
}
```

### クラウドセッションでも有効にする(project scope)

user scope の `enabledPlugins` は**クラウドに引き継がれない**。クラウドでも効かせたいリポジトリでは、同じ内容を**リポジトリの `.claude/settings.json`(git 追跡対象)**に書く。

## 制約

- output style は**メイン会話にのみ適用され、サブエージェントには適用されない**(fork は例外)
- プラグインで配れるのは skills / agents / commands / output-styles / hooks / MCP サーバー等であって、`permissions` や `env` などの settings キーは配れない(プラグイン内 `settings.json` は `agent` と `subagentStatusLine` のみサポート)
