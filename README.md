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

## トラブルシュート

### 有効にしたのに口調が適用されない

**症状**: `enabledPlugins` に入れてセッションを起動したのに応答が通常文体のまま。`/config` の Output style にビルトイン(`default` / `Proactive` / `Explanatory` / `Learning`)しか出てこない。

**原因の切り分け**: 設定ファイルではなく、**インストールの実体**を疑う。`installed_plugins.json` は「インストール済み」と記録しているのに、そこに書かれた `installPath` のディレクトリだけが存在しない、という状態になりうる。

```sh
# 記録上の installPath を読む
python3 -c "import json,os;d=json.load(open(os.path.expanduser('~/.claude/plugins/installed_plugins.json')));print(d['plugins']['tachikoma-tone@toshi0607-plugins'])"

# その実体があるか確認する(無ければこれが原因)
ls ~/.claude/plugins/cache/toshi0607-plugins/tachikoma-tone/*/output-styles/
```

**復旧**: 同じ scope で入れ直すと実体が再取得される。

```sh
claude plugin install tachikoma-tone@toshi0607-plugins --scope user
```

**紛らわしい点**: `~/.claude/plugins/marketplaces/toshi0607-plugins/plugins/tachikoma-tone/output-styles/tachikoma.md` は marketplace のクローンとして残っているため、`find` で探すと**ファイルが有るように見える**。ローダーが読むのは `cache/` 配下の installPath なので、marketplace 側の存在は判断材料にならない。

**確認するとき**: output style はセッション開始時にシステムプロンプトへ読み込まれる。復旧しても実行中のセッションには反映されないので、新しいセッションを起動して確認する。

### main を更新したのに反映されない

**症状**: main に口調ルールの変更をマージしたのに cache の中身が古いまま。`claude plugin update` は「already at the latest version」と言って何もしない。

**原因は 2 段**:

1. **marketplace クローンが古い**。公式以外の marketplace は `autoUpdate` が既定で無効なので、`~/.claude/plugins/marketplaces/` のクローンは追加時点のまま止まる。まず手動で更新する。

   ```sh
   claude plugin marketplace update toshi0607-plugins
   ```

2. **`plugin.json` の `version` がキャッシュキー**。`version` を設定している場合、その文字列を上げない限り新しいコミットは配信されない(公式仕様: バージョン固定)。中身を変えるコミットでは `version` も上げること。

**ワークアラウンド**: `version` を上げられない事情があるときは、同一 version のまま入れ直せば marketplace クローンの現物が cache にコピーし直される。

```sh
claude plugin uninstall tachikoma-tone@toshi0607-plugins
claude plugin install tachikoma-tone@toshi0607-plugins --scope user
```

### `outputStyle` を手で指定したいとき

`force-for-plugin: true` があるので通常は不要だが、明示したい場合に指定する名前は **output style の frontmatter の `name`(`tachikoma`)** であって、**プラグイン名(`tachikoma-tone`)ではない**。

```json
{ "outputStyle": "tachikoma" }
```

なお `/config outputStyle=...` のヘルプに列挙されるのはビルトインのみで、プラグイン由来のスタイル名は出てこない。ここに出ないことは「読み込まれていない」証拠にはなるが、正しい名前を知る手段にはならない。

## 制約

- output style は**メイン会話にのみ適用され、サブエージェントには適用されない**(fork は例外)
- プラグインで配れるのは skills / agents / commands / output-styles / hooks / MCP サーバー等であって、`permissions` や `env` などの settings キーは配れない(プラグイン内 `settings.json` は `agent` と `subagentStatusLine` のみサポート)
