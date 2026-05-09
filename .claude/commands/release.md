---
allowed-tools: Bash, Edit, Read
description: "carta リポジトリのリリース（version bump → tag → GitHub Release）"
---

# /release

このリポジトリ（carta）自身のリリース作業をローカルで一気通貫に実行する。npm publish や CI workflow は無く、配布は GitHub の tag + marketplace 経由のため、必要な作業は以下に集約される:

1. `.claude-plugin/plugin.json` と `.claude-plugin/marketplace.json` の version をバンプ
2. CHANGELOG.md を更新
3. commit / tag / push
4. GitHub Release 作成
5. ローカル Claude Code の plugin cache を新バージョンに追従

## 引数

`$ARGUMENTS` でバージョンを指定できる（省略時はコミット履歴から自動判定）:

- `/carta:release` — Conventional Commits から自動判定
- `/carta:release 0.2.0` — 指定バージョンで固定

## 実行ポリシー

- **operational task（運用作業）**。サブエージェントは spawn しない
- 失敗時は該当ステップだけやり直す（全体リトライ不要）
- リリース対象は **main ブランチ**。worktree 内で実行している場合は `cd "$PROJECT_ROOT"` で main 側に移動してから編集・commit する

## 手順

### 0. preflight（プラグインマニフェスト検証）

`.claude-plugin/plugin.json` と `.claude-plugin/marketplace.json` の整合性・スキーマ準拠を確認する。`claude plugin validate` が落ちる場合は manifest の構造的問題なので先に直す:

```bash
cd "$PROJECT_ROOT"
claude plugin validate . || { echo "plugin validate failed; aborting" >&2; exit 1; }

PLUGIN=$(node -p "require('./.claude-plugin/plugin.json').version")
MARKET=$(node -p "require('./.claude-plugin/marketplace.json').plugins.find(p => p.name === 'carta').version")
echo "versions — plugin=$PLUGIN marketplace=$MARKET"
if [ "$PLUGIN" != "$MARKET" ]; then
  echo "WARNING: plugin.json と marketplace.json の version が不一致。バンプ時に揃える" >&2
fi
```

### 1. 現在のバージョンとコミット履歴を取得

```bash
cd "$PROJECT_ROOT"
CURRENT=$(node -p "require('./.claude-plugin/plugin.json').version")
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
if [ -n "$LAST_TAG" ]; then
  COMMITS=$(git log ${LAST_TAG}..HEAD --oneline)
else
  COMMITS=$(git log --oneline -20)
fi
echo "current=$CURRENT  last_tag=$LAST_TAG"
```

### 2. バージョンを判定

`$ARGUMENTS` が指定されていればそれを `NEW_VERSION` とする。未指定なら Conventional Commits で判定:

| キーワード | 変更レベル |
|---|---|
| `BREAKING CHANGE`, `!:` | major |
| `feat:`, `feat(` | minor |
| `fix:` / `chore:` / `docs:` のみ | patch |

コミット群で最も大きい変更レベルを採用。

### 3. CHANGELOG.md を更新（無ければ新規作成）

```bash
cd "$PROJECT_ROOT"
if [ ! -f CHANGELOG.md ]; then
  cat > CHANGELOG.md <<'HEADER'
# Changelog

All notable changes to this project will be documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

HEADER
fi
```

その上でヘッダ直下に新バージョンブロックを追記する:

```
## [X.Y.Z] - YYYY-MM-DD

### Added
- 新機能の説明

### Changed
- 変更の説明

### Fixed
- 修正の説明
```

**分類:** `feat:` → Added / `fix:` → Fixed / それ以外 → Changed。コミットメッセージそのままコピーせず、ユーザーが読んで意味がわかる説明に書き直す。

### 4. version をバンプ（plugin.json + marketplace.json）

```bash
cd "$PROJECT_ROOT"

node -e "const p=require('./.claude-plugin/plugin.json'); p.version='${NEW_VERSION}'; require('fs').writeFileSync('.claude-plugin/plugin.json', JSON.stringify(p,null,2)+'\n')"

node -e "const m=require('./.claude-plugin/marketplace.json'); const t=m.plugins.find(p=>p.name==='carta'); if(!t){throw new Error('carta plugin not found in marketplace.json')} t.version='${NEW_VERSION}'; require('fs').writeFileSync('.claude-plugin/marketplace.json', JSON.stringify(m,null,2)+'\n')"
```

### 5. commit / tag / push

```bash
cd "$PROJECT_ROOT"
git add CHANGELOG.md .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "chore: release v${NEW_VERSION}"
git tag "v${NEW_VERSION}"
git push origin main
git push origin "v${NEW_VERSION}"
```

### 6. GitHub Release を作成

CHANGELOG.md から該当バージョンのセクションを抽出して Release notes にする:

```bash
cd "$PROJECT_ROOT"
NOTES=$(awk "/^## \[${NEW_VERSION}\]/{found=1; next} /^## \[/{if(found) exit} found" CHANGELOG.md)
gh release create "v${NEW_VERSION}" --title "v${NEW_VERSION}" --notes "$NOTES"
```

### 7. ローカル plugin cache を新バージョンに追従

GitHub に tag が打たれても Claude Code のローカル plugin キャッシュは手動更新しない限り古いまま。`claude plugin` CLI で反映させる:

```bash
# 7.1 marketplace のソースキャッシュを最新 main に追従
if claude plugin marketplace list 2>/dev/null | grep -q "hummer98-carta"; then
  claude plugin marketplace update hummer98-carta
else
  echo "marketplace hummer98-carta not registered locally; skipping"
fi

# 7.2 user scope にインストール済みなら新バージョンへ更新
if claude plugin list 2>/dev/null | grep -q "carta@hummer98-carta"; then
  claude plugin update carta@hummer98-carta
  echo "Claude Code のリスタート推奨: SessionStart hook と bin/ を再読込するため"
else
  echo "plugin not installed via marketplace; skipping"
fi
```

### 8. bin/ を変更したリリースの場合の検証

`bin/cmux-*` `bin/cfork` 等を更新した場合、**現セッション内で動作確認してはいけない**。PATH がセッション開始時の旧バージョンに固定されているため、修正済みのスクリプトを呼んでいるつもりで旧版を実行してしまう。

正しい検証:

1. §7 で `claude plugin update` 完了
2. 新しい cmux ウィンドウまたはワークスペースを開く
3. 新セッション内で `which cmux-send-key` が新バージョンの `bin/` を指すことを確認
4. 別 surface を作って動作テスト

詳細は CLAUDE.md の「bin/ 更新の検証手順」を参照。

### 9. 完了報告

```
リリース完了: v${CURRENT} → v${NEW_VERSION}
- tag: v${NEW_VERSION}
- CHANGELOG.md: 更新済み
- plugin.json / marketplace.json: 更新済み
- GitHub Release: 作成済み
- local plugin cache: 更新済み（要 Claude Code リスタート）
```

## 注意事項

- carta は npm パッケージではないため `package.json` も `npm publish` も無い
- 配布は GitHub の tag + marketplace 経由のみ。CI workflow は使わない
- リリースコミット/タグは **main ブランチに直接打つ**（PR 経由にしない）
- バージョン更新対象は `.claude-plugin/plugin.json` と `.claude-plugin/marketplace.json` の **2 箇所のみ**

<!-- 参考元: hummer98/using-cmux/.claude/commands/release.md -->
