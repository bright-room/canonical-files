# canonical-files

repository-fanout が各リポジトリへ配布する共通ファイルの**正本**。
worker の `TEMPLATES_REPO` 変数(`bright-room/canonical-files`)からこのリポのルートが読まれる。
エンジン本体は [bright-room/repository-fanout](https://github.com/bright-room/repository-fanout)。

## 構成

```
canonical-files/
├── catalog.json             # 全管理ファイルの唯一の宣言。ここに無いパスへの寄与はエラー
├── profiles/<name>/         # profile ごとの寄与データ（旧 base/languages/bundles を統一）
│   └── contributes.json
└── templates/                # 全ファイルの本文（Liquid）
    └── *.liquid
```

### catalog.json

配布対象の全パスをここで宣言する。1 パスにつき:

- `file_type` — `text` / `json` / `yaml` / `markdown` など
- `mode` — `replaced`（ファイル全体を収束）/ `create-only`（無ければ配置、あれば触らない）/ `managed`（構造化マージ。`managed_paths` でキー単位のマージ方法を指定)

**catalog.json に無いパスへの寄与はエラーになる。**

### profiles/\<name\>/contributes.json

`base` / 各言語(`typescript` / `terraform` / `java` / `kotlin` / `go` / `python` / `rust`) / `oss`(公開リポ向け定型ドキュメント)がそれぞれ 1 profile。リポが宣言した profile の寄与データが catalog.json のパスごとにマージされる。

`template` キーはそのファイルの本文テンプレート(`templates/` 配下のファイル名)を指すと同時に、**そのファイルを配布するトリガー**でもある。1 パスにつき、宣言している profile のうち**ちょうど 1 つ**が `template` を持つ必要がある。

### templates/

全ファイルの本文を Liquid テンプレートとして置く。

- `{{ contributions.* }}` — 寄与データ(profiles の contributes.json)をマージした結果
- `{{ contents.* }}` — 配布先リポ個別の値(tf 側 `fanout.contents`)

## ファイルを追加するには

Worker 側の変更は不要。以下の 3 点だけで完結する。

1. `catalog.json` に配布先パスのエントリを 1 つ追加する
2. 本文が必要なら `templates/` にテンプレートを 1 枚置く
3. 配布トリガーにしたい profile の `contributes.json` に該当パスの行を 1 つ追加する(`template` キーで上記テンプレートを指定)

## 検証

PR ごとに CI(repository-fanout の `cli validate`)が catalog の検証と全 profile の描画スモークを実行する。詳細な設計・マージ挙動は repository-fanout の
`docs/superpowers/specs/2026-07-05-catalog-profiles-design.md` を参照。

## 注意

- `templates/gitignore.liquid` の正準形は repository-fanout の core テスト(`GITIGNORE_LIQUID`)とバイト一致で固定されている。勝手に整形しない。
- GitHub Actions のワークフローファイルなど `${{ }}` を含むファイルを将来配る場合は、catalog.json 側でそのパスに `raw: true` を指定して Liquid 描画をスキップする。

## 設計ガードレール(2026-07 全リポ実態調査に基づく)

テンプレに**入れてはいけない**もの:

- `.claude/` の丸ごと ignore — `.claude/rules/` `.claude/skills/` `.claude/settings.json` を意図的にコミットするリポが複数ある。個人ローカル分(`settings.local.json` 等)のみ base で ignore する。
- `Cargo.lock` / `.terraform.lock.hcl` — 全リポでコミット運用(terraform は CI が `-lockfile=readonly` 前提)。
- `mise.toml` / `.tool-versions` を捕捉するパターン — ツールバージョンのピン留めとしてコミット運用。
- `.envrc` — コミット運用のリポがある(direnv 設定は共有物になりうる)。機密の防波堤は `.env` 系で足りる。
- 汎用 `*.local` — chezmoi の命名規約(dotfiles)と衝突する。env 系は `.env.local` / `.env.*.local` の具体パターンに留める。
- kotlin への `.editorconfig` 配布 — 各リポが ktlint ルールをリポ固有に持つため replace で壊れる。
