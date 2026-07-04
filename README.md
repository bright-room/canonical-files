# canonical-files

repository-fanout が各リポジトリへ配布する共通ファイルの**正本**。
worker の `TEMPLATES_REPO` 変数(`bright-room/canonical-files`)からこのリポのルートが読まれる。

## 構成

```
canonical-files/
├── strategies.json          # 配布先パス→戦略の map（必須。不在/不正だと fanout が fail fast）
├── base/                    # 全リポに常時適用される言語非依存の単位
│   ├── fragment.json        #   非ファイル貢献（renovate extends / gitignore セクション）
│   └── files/               #   配布ファイル（renovate.json / .gitignore / CODEOWNERS / release.yml）
├── languages/<lang>/        # リポが languages に宣言した時だけ適用
│   ├── fragment.json        #   その言語の貢献宣言
│   └── files/               #   （任意）言語別の配布ファイル
├── bundles/<name>/          # リポが bundles に宣言した時だけ適用（言語と独立な opt-in 束）
│   └── ...                  #   構造・マージ意味論は languages と同一（違いは分類のみ）
└── seeds/                   # create-only（無ければ配置、あれば触らない）。当面なし
```

言語: `typescript` / `terraform` / `java` / `kotlin` / `go` / `python` / `rust`
束: `oss`（公開リポ向け定型ドキュメント CONTRIBUTING.md / SECURITY.md）

## sync 戦略

| 戦略 | 対象 | 反映方法 |
|---|---|---|
| replace | 既定（`release.yml`、`.editorconfig` 等） | ファイル全体を収束 |
| create-only | `seeds/**` | 無ければ配置、あれば触らない |
| managed-block | `.gitignore`、`.github/CODEOWNERS` | ブロック内だけ更新（リポ独自ルールはブロック外に温存） |
| extends-field | `renovate.json` | `extends` の管理エントリだけ更新（他キーは不可侵） |

パス→戦略の割り当てはルートの **`strategies.json`** で宣言する（値は `extends-field` / `managed-block` のみ。未登録パスは replace、`seeds/**` は create-only）。

`{{gitignore}}` / `{{renovate_extends}}` / `{{codeowner}}` は fanout が描画するプレースホルダ。
このリポ自身には配布しない（`base/files/**` はテンプレとして扱う）。

## fragment.json

```json
{
  "renovate": ["github>bright-room/renovate-config:<lang>"],
  "gitignore": [
    { "section_comment": "見出し（`### 見出し ###` 形式で描画される）", "ignores": ["パターン", "..."] }
  ]
}
```

- renovate preset の本体は [bright-room/renovate-config](https://github.com/bright-room/renovate-config)（言語名 = preset 名の 1:1）。**python は preset 未整備のため gitignore 貢献のみ**。
- gitignore セクションは base → 宣言 languages → 宣言 bundles の順に連結され、パターンは横断で dedup（初出優先）。
- 言語を増やす: renovate-config に同名 preset を用意し `languages/<lang>/fragment.json` を置くだけ。束を増やすのも同様に `bundles/<name>/` を置くだけ。

## 設計ガードレール（2026-07 全リポ実態調査に基づく）

テンプレに**入れてはいけない**もの:

- `.claude/` の丸ごと ignore — `.claude/rules/` `.claude/skills/` `.claude/settings.json` を意図的にコミットするリポが複数ある。個人ローカル分（`settings.local.json` 等）のみ base で ignore する。
- `Cargo.lock` / `.terraform.lock.hcl` — 全リポでコミット運用（terraform は CI が `-lockfile=readonly` 前提）。
- `mise.toml` / `.tool-versions` を捕捉するパターン — ツールバージョンのピン留めとしてコミット運用。
- `.envrc` — コミット運用のリポがある（direnv 設定は共有物になりうる）。機密の防波堤は `.env` 系で足りる。
- 汎用 `*.local` — chezmoi の命名規約（dotfiles）と衝突する。env 系は `.env.local` / `.env.*.local` の具体パターンに留める。
- kotlin への `.editorconfig` 配布 — 各リポが ktlint ルールをリポ固有に持つため replace で壊れる。

詳細な設計・マージ挙動は repository-fanout の
`docs/superpowers/specs/2026-06-26-repository-fanout-design.md` と `docs/superpowers/specs/sample/` を参照。
