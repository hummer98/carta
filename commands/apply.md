---
allowed-tools: Read, Edit, Write, Bash
description: "現在のプロジェクトの CLAUDE.md に carta 憲法を適用 / 同期する"
argument-hint: "[<path>] [--force]"
---

# /carta:apply

カレントリポジトリ（または指定パス）の `CLAUDE.md` に最新の carta 憲法を適用する。マーカー方式で **既存内容を保護** しつつ、`<!-- carta:begin -->` ～ `<!-- carta:end -->` の中身だけを最新版に同期する。

## 引数

`$ARGUMENTS`:
- 指定なし: カレントディレクトリ（git root を優先、無ければ `pwd`）の `CLAUDE.md` を対象にする
- `<path>`: 指定パス。ディレクトリなら `<path>/CLAUDE.md`、ファイルなら直接そのパス
- `--force`: マーカー内本文の sha が記録値と不一致（手で編集された痕跡）の場合でも上書きを許可する

## 安全要件（必ず守る）

1. **書き込み前に必ず diff をユーザーに提示し、承認を得てから書き込む**。一発適用しない。
2. **マーカー外のテキストは 1 byte も変更しない**。書き込み後に「マーカー外文字列が apply 前と完全一致するか」を再確認する。
3. **マーカーが破損している（begin/end の片方しか無い、順序が逆、複数ある）場合は自動修復せず、エラーで停止する**。ユーザーに手動修復を促す。
4. **マーカー内 sha 不一致は `--force` 無しでは上書きしない**。

## 手順

### Step 1. 憲法本体の取得とハッシュ算出

```bash
ROOT="${CLAUDE_PLUGIN_ROOT:-$(pwd)}"
SRC="$ROOT/assets/constitution.md"
[ -f "$SRC" ] || { echo "constitution.md not found at $SRC" >&2; exit 1; }

# SHA-256 短縮 hash（先頭 7 文字）
HASH=$( (sha256sum "$SRC" 2>/dev/null || shasum -a 256 "$SRC") | awk '{print $1}' )
SHORT_HASH=${HASH:0:7}

VERSION=$(node -p "require('$ROOT/.claude-plugin/plugin.json').version")
echo "carta v$VERSION sha=$SHORT_HASH"
```

### Step 2. 対象 CLAUDE.md の特定

```bash
ARG_PATH="<引数で渡された path、無ければ空>"
if [ -z "$ARG_PATH" ]; then
  TARGET_DIR=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  TARGET="$TARGET_DIR/CLAUDE.md"
elif [ -d "$ARG_PATH" ]; then
  TARGET="$ARG_PATH/CLAUDE.md"
else
  TARGET="$ARG_PATH"
fi
echo "target: $TARGET"
```

### Step 3. 既存 CLAUDE.md の状態判定

`TARGET` を読み、以下の 5 ケースに分岐:

| 状態 | 判定方法 | 次のステップ |
|------|---------|------------|
| 存在しない | `[ ! -f "$TARGET" ]` | Step 4a 新規作成 |
| 存在するがマーカー無し | begin / end の grep ヒット数がいずれも 0 | Step 4b 冒頭挿入 |
| マーカー両方あり | begin / end ともにヒット 1 | Step 4c 同期判定 |
| マーカー破損 | begin / end のヒット数が不揃い、順序逆転、または複数 | エラーで停止（手動修復を促す） |

検出例:

```bash
BEGIN_COUNT=$(grep -cE '^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+ -->$' "$TARGET" || true)
END_COUNT=$(grep -cE '^<!-- carta:end -->$' "$TARGET" || true)

if [ "$BEGIN_COUNT" != "$END_COUNT" ] || [ "$BEGIN_COUNT" -gt 1 ]; then
  echo "ERROR: marker broken (begin=$BEGIN_COUNT end=$END_COUNT). 手動で修復してください" >&2
  exit 1
fi
```

### Step 4a. 新規作成

```
<!-- carta:begin v{VERSION} sha={SHORT_HASH} -->
{constitution body}
<!-- carta:end -->

## プロジェクト固有

（リポジトリ固有の規約・運用・テストコマンド等をここに記述）
```

新規作成の場合は diff 提示の代わりに「新規ファイルの内容」を見せて承認を取る。

### Step 4b. 既存 CLAUDE.md の冒頭にマーカーブロックを挿入

既存内容の前に以下を挿入（最後に空行 1 行を入れる）:

```
<!-- carta:begin v{VERSION} sha={SHORT_HASH} -->
{constitution body}
<!-- carta:end -->

```

既存内容は完全保持（マーカー外保護）。

### Step 4c. 同期判定

既存マーカーから version と sha を抽出し、本文も切り出す:

```bash
EXIST_VERSION=$(awk 'match($0, /<!-- carta:begin v([^ ]+) sha=([0-9a-f]+) -->/){
  s = substr($0, RSTART, RLENGTH);
  sub(/.*v/, "", s); sub(/ .*/, "", s);
  print s; exit
}' "$TARGET")

EXIST_SHA=$(awk 'match($0, /<!-- carta:begin v([^ ]+) sha=([0-9a-f]+) -->/){
  s = substr($0, RSTART, RLENGTH);
  sub(/.*sha=/, "", s); sub(/ -->/, "", s);
  print s; exit
}' "$TARGET")

EXIST_BODY=$(awk '/^<!-- carta:begin /{flag=1; next} /^<!-- carta:end -->$/{flag=0} flag' "$TARGET")
```

判定:

- **すべて一致** （`EXIST_VERSION == VERSION` かつ `EXIST_SHA == SHORT_HASH` かつ `EXIST_BODY` が新版本文と一致） → 「最新です」と報告して終了。書き込まない（idempotent）。
- **EXIST_BODY と新版本文は一致するが EXIST_SHA が記録 hash と違う** → これは矛盾なので通常起こらないが、検出したら警告。
- **EXIST_BODY が新版本文と異なるのに、記録された sha と現実の `EXIST_BODY` の sha が一致しない**（マーカー内が手で改変された痕跡） → 警告を出し、`--force` が指定されていれば上書き、無ければ停止。
- **それ以外（バージョンや本文が異なる、新版へ更新が必要）** → diff を提示して承認を得てから上書き。マーカーの version/sha も更新。

### Step 5. diff 提示と承認

書き込み前に、変更の要約を以下の形式で提示:

```
[carta:apply] {target}
  before: v{EXIST_VERSION} sha={EXIST_SHA}  ({X} 行)
  after:  v{VERSION} sha={SHORT_HASH}        ({Y} 行)
  --- diff (マーカー内のみ) ---
  ...
```

ユーザーが承認したら次のステップへ進む。

### Step 6. 書き込みとマーカー外保護のアサーション

書き込み後、以下を検証:

1. begin / end マーカーが各 1 行ずつ存在する
2. マーカー外（begin より前のテキスト + end より後のテキスト）が apply 前と完全一致する
3. 新しいマーカーの version / sha が想定値と一致する

不一致なら直ちにロールバック（書き込み前のテキストを書き戻す）してエラー報告。

### Step 7. 完了報告

```
[carta:apply] OK
  target:  {TARGET}
  version: v{EXIST_VERSION} → v{VERSION}   （新規時は v- → v{VERSION}）
  sha:     {EXIST_SHA} → {SHORT_HASH}
  changes: {summary}
```

## エッジケース

| 状況 | 挙動 |
|------|------|
| `assets/constitution.md` が見つからない | エラーで停止。`${CLAUDE_PLUGIN_ROOT}` の確認を促す |
| 対象 CLAUDE.md が読めない（パーミッション） | エラーで停止 |
| 既存マーカーが複数ある | エラーで停止（v0.1.0 では root の 1 つだけが対象） |
| 既存マーカーの順序が逆（end が先・begin が後） | マーカー破損として停止 |
| version 文字列が不正 | エラーで停止（`<!-- carta:begin vX.Y.Z sha=abcdefg -->` を期待） |
| 改行コードが CRLF の CLAUDE.md | LF に正規化せず、書き込み時も既存の改行コードを保持する（マーカー外保護の一環） |

## 実装メモ

- shell + Read / Edit / Write の組み合わせで実現。複雑なテンプレートエンジンは使わない
- マーカー判定の正規表現は `<!-- carta:begin v(\S+) sha=([0-9a-f]{7}) -->` を期待。揺らぎは許容しない
- `git rev-parse --show-toplevel` が空（git 未初期化）の場合は `pwd` にフォールバック
- 大文字小文字は厳密に一致させる（`Carta:Begin` 等は別物として扱う）
